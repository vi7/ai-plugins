# Don't put an LLM where a bash script will do

*Or: how I almost gave a language model a login, a service account and a cron job to check four numbers.*

Recently I was debugging a production Prometheus exporter that was restarting during the night. It wasn't crashing in the usual sense. It was getting killed by Kubernetes because its liveness probe timed out.

The reason was fairly simple: the exporter depended on another service that became slow when the nightly jobs started. The probe had a short timeout, so Kubernetes eventually decided the exporter was dead and restarted it.

The fix was boring: increase the probe timeout, adjust the monitoring timeout, deploy the change.

The interesting question was what happens after the deployment.

The fix only really matters at 3 a.m. That's when the nightly load happens. I didn't want to wake up at 3 a.m. just to check whether the exporter survived. So I needed some kind of automated verification.

## The obvious AI solution

I had been using an AI agent during the investigation and it was actually quite useful. It helped correlate the restarts with the nightly load, look through metrics and logs, find a problem with one of my assumptions, and prepare the ticket and merge requests.

So the next idea seemed reasonable:

> Run the agent every morning for a few days and ask it whether the fix worked.

The agent could query the Prometheus, look at the deployment and send me a message. Then I started thinking about what would actually be required to run an agent unattended.

It would need:

* a dedicated non-root user;
* its own authentication to the model provider;
* read-only access to the monitoring system;
* some kind of K8S credentials if it needed its data;
* a restricted tool set;
* a timeout and a way to stop it;
* logs;
* and probably some tests around the whole thing.

That's a lot of infrastructure for a question that turned out to be:

1. Did the exporter restart?
2. Was it up during the whole observation period?
3. What was the maximum successful scrape duration?
4. Did the alert fire?

That's four PromQL queries and four comparisons.

At that point I stopped.

## The boring solution

I wrote a shell script instead. It was around 80 lines, most of which were notification and error handling. The actual check was tiny.

Something like this:

```bash
query() {
    curl -sk --max-time 25 -G "$PROM/api/v1/query" \
        --data-urlencode "query=$1" |
        jq -r '
            select(.status == "success") |
            .data.result[0].value[1] // "none"
        '
}

check() {
    local name="$1"
    local query="$2"
    local condition="$3"

    local value
    value=$(query "$query") || {
        errors+=("$name: query failed")
        return
    }

    [[ "$value" == "none" ]] && {
        errors+=("$name: no data")
        return
    }

    jq -en --arg v "$value" \
        '($v | tonumber) as $v | '"$condition" >/dev/null ||
        failures+=("$name = $value")
}

check restarts \
    '...' \
    '$v == 0'

check uptime \
    '...' \
    '$v == 1'

check scrape_time \
    '...' \
    '$v < 25'
```

The real script had a few more checks and notification handling, but the principle was the same.

There were three possible results:

**CONFIRMED**

Everything was within the expected limits.

No notification during the observation period. On the last day, send one short confirmation.

**REGRESSED**

Something went wrong.

Send me a message with the relevant result and the rollback procedure.

**INCONCLUSIVE**

The monitoring query failed, there was no data, or the thing being checked no longer existed.

Send a message too.

This last case is important:

**Silence should mean "checked and everything is fine", not "the checker failed".**

So I also wrote one JSONL record for every run. Nothing fancy, just enough to know that the check actually happened.

### One useful detail

The script also checked that the deployment being verified was still the one I intended to verify.

I used an identifier associated with the newly deployed workload rather than simply asking Prometheus about "the exporter".

That matters because deployments can change while your verification job is waiting.

If somebody rolls back or deploys another version, the original workload disappears. In that case the script reports `INCONCLUSIVE` instead of accidentally verifying the new deployment.

It's a small detail, but it prevents a particularly annoying kind of false positive.

## systemd instead of cron

I could have used cron, but systemd made the rest of this surprisingly simple.

The service looked roughly like this:

```ini
[Unit]
ConditionPathExists=!/etc/example/PAUSE

[Service]
Type=oneshot
User=example-verify
ExecStart=/usr/local/libexec/example-verify/verify.sh
TimeoutStartSec=300

StateDirectory=example-verify
LoadCredential=notifier.env:/path/to/notifier.env

ProtectSystem=strict
ProtectHome=yes
PrivateTmp=yes
NoNewPrivileges=yes
```

And the timer contained the few dates on which I wanted the check to run:

```ini
[Timer]
OnCalendar=...
OnCalendar=...
OnCalendar=...

Persistent=true
```

There were a couple of nice things here.

- `LoadCredential=` lets systemd provide the notification credential to the service without copying it into the script's environment or configuration.

- `ConditionPathExists=!PAUSE` gives me a simple kill switch. Create the file and systemd doesn't start the service.

And because the timer had a finite number of scheduled runs, it didn't become another forgotten cron job that would still be running six months later ;)

There was also a small systemd gotcha. I initially used:

```ini
RuntimeMaxSec=300
```

because that's what I wanted: kill the process after five minutes. But it appeared that if service is `Type=oneshot`, that isn't the right setting for limiting its startup/run phase.

The correct option here was:

```ini
TimeoutStartSec=300
```

`systemd-analyze verify` happily caught it. It's worth running, costs nothing.

## Where the LLM actually helped

This isn't an anti-LLM story, on the contrary the LLM was useful during the investigation.

The interesting part was figuring out **what should be checked**.

For example, one of the initial metrics I was looking at seemed to show that the exporter had enough headroom, while second review showed that this conclusion was wrong.

The apparent maximum duration was based on successful scrapes. When a scrape exceeds the timeout, it can fail instead of giving you a nice metric saying "this scrape took 27 seconds". So a graph showing a maximum of 9 seconds doesn't necessarily mean the exporter never took longer than 9 seconds. It can mean that 9 seconds was the longest successful scrape.

The failed scrapes were the important part. That kind of investigation is where an LLM is useful:

* correlate different signals;
* suggest what to look at;
* challenge assumptions;
* generate queries;
* review a proposed solution;
* write the boring documentation and ticket text.

## My current rule

I ended up with a fairly simple rule:

> If the pass/fail condition can be written as a comparison, use a script. / If you still need to figure out what should be compared, an LLM can be useful.

The important part is not "bash good, AI bad". It's rather about using the right amount of machinery.

An LLM is useful when there is ambiguity and you need judgement. A script is useful when the decision has already been made and needs to be repeated reliably (not "re-decided each and every time").

In this case, using an agent would have meant building a small security boundary around a system whose job was basically:

```text
query
query
query
query
compare
notify
```

The shell script was easier to understand, easier to secure and easier to debug.

And it was considerably cheaper ;)

## One last surprise

After three nights the check reported exactly what I wanted: no restarts and scrape times comfortably below the new timeout. So I deleted the timer, the service, the script and the temporary user.

Except I didn't.

The cleanup command appeared to run successfully but didn't remove the files. The reason was my shell configuration.

My interactive shell had an `rm -i` alias. The non-interactive environment inherited it. Without a terminal to answer the confirmation prompt, `rm -i` effectively did nothing.

That was a good reminder that automation doesn't necessarily run in a clean environment. Your shell configuration is part of the environment too.

If you use interactive-only aliases, make sure they are actually limited to interactive shells:

```bash
if [[ $- == *i* && -z ${AGENT_SHELL:-} ]]; then
    alias rm='rm -i'
fi
```

Use whatever environment variable or detection mechanism makes sense for your agent.

The broader lesson was probably the same as the one from the beginning:

**Use the clever tool to figure out what needs to be checked. Use the boring tool to check it.**