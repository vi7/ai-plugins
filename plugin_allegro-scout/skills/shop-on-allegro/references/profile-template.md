# Category profiles: reuse the universal basket engine

A later category-specific skill/plugin may add this minimal profile (human-readable markdown is enough):

- Profile name and use cases: e.g. `household-chemicals` for monthly detergent restock.
- Slot-normalization: what identifies an interchangeable item? brand, EAN, purpose, ingredients, exact dimensions, package size or machine compatibility.
- Non-negotiable checks: risks/contraindications, certifications only if required, expiry/date/batch, seller or storage constraints.
- Price normalization: per kg, liter, tablet, effective dose, number of uses, covered square meters, etc. State any assumptions needed for concentration/dilution.
- Allowed substitution policy: strict brand/model, equivalent composition, or general outcome. A profile must not replace user-specified exact SKUs without permission.
- Preferences: user's real preferred brands, known allergies/sensitivities, scents, package sizes or procurement cadence only when given. Avoid extrapolation from the profile category.
- Volume and stocking limits: don't overbuy for free shipping; consider shelf-life, storage, usage and disposal.
- Quality evidence: identify specific sources or product label information necessary to verify claims.

Example profiles:
- **Household chemicals**: purpose, dilution/dose, fragrance, chemical concentration, surface compatibility, closure/refill, storage and hazard label. Unit price per real use rather than raw bottle price.
- **Fertilizers**: N-P-K/micronutrients, product formulation and concentration, target crop, season, area, dose and shelf life. Don't treat differently formulated products as equivalent solely by weight.
- **Consumables**: device exact model, OEM/compatible status and part number, pack size, failure risk, warranty/compatibility claims.
- **Pantry**: ingredients, origin if relevant, package integrity, expiry, roasting/salt status, unit price, avoid forced bulky lots.

Keep the main `shop-on-allegro` skill unchanged when adding a profile unless a real generic bug was discovered. Profiles should use the same delivery/seller/uncertainty math and the same compact output. Implement a distinct routing description so the profile is invoked only for its class of items.
