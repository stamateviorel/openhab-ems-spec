# Tasks

## 1. Alignment
- [ ] 1.1 Inventory existing forecast producers (solar forecast binding, weather bindings) and their series shapes
- [ ] 1.2 Agree the provider surface with those maintainers

## 2. Specification hardening
- [ ] 2.1 Define units/quantities per forecast kind (Power, Temperature, ratio)
- [ ] 2.2 Specify derived-forecast registration (who computes, when it refreshes)

## 3. Prototype path
- [ ] 3.1 One real source behind the surface end-to-end (series → chart overlay → engine read)
- [ ] 3.2 Heating-need derivation as a pure, testable function over temp + solar series
