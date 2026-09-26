# Design Deviations

Changes made during implementation compared to the Designathon submission, with reasons.

Design file: [LegendX_Designathon (Figma)](https://www.figma.com/design/dtE7b7R8CPwZh5scsxu6zx/LegendX_Designathon?node-id=16-2). The baseline below lists the decisions in that design, so every departure can be compared against it.

## Departures

| Screen / Flow | Designathon | Implemented | Reason |
|---|---|---|---|

## Designathon baseline (the Day 5 design)

### Product and roles
| Decision | Detail |
|---|---|
| App type | One responsive web app, built as a PWA: installs on the phone, works offline, uses the camera. No native apps |
| Sign-in | One sign-in for the whole app; the account's role decides which screens appear. Loaders sign in with a 4-digit PIN on the shared dock tablet |
| Dispatcher | Desktop 1440, light theme by default |
| Store manager | Phone 412 and desktop 1440 (counter), light by default. Desktop reuses the phone screens' content |
| Loader | Shared tablet 1280 × 800, **dark by default** (02:00–04:00 shifts). The same screens at phone size, because loaders are judged on phone-sized screens |
| Driver | Phone 412, follows the phone's theme. Designed for use when safely stopped: one stop per screen, one main action |
| Themes | Every role has Light and Dark, with a Sun/Moon toggle in every top bar; the choice is saved per user (per PIN on the loader tablet) |

### Screens (39, each in Light and Dark)
| Role | Screens |
|---|---|
| Sign in | Phone · Desktop |
| Store manager (phone and desktop) | Place order · Tomorrow (deferral notice, the "Keep deferred" branch) · Notifications · Confirm receipt · Proof of delivery · Report issue · Receipt confirmed · History |
| Dispatcher | Tomorrow's plan · Deferred-order sheet · Plan published · Live · Issues |
| Loader (tablet and phone) | PIN sign-in · Load list · Flag shortfall · Vehicle ready |
| Driver | Run · Stop · Signature · Delivered · Report problem |
| Degradation | Driver offline · Dispatcher live with an offline driver · Back online with a conflict |

The prototype has 11 flows; none jumps between devices or themes.

### Planning and deferrals
| Decision | Detail |
|---|---|
| Who decides who waits | The solver proposes a plan inside all 7 rules; the dispatcher decides deferrals (core tradeoff, Figma page 06) |
| Skip history | The deferred-order sheet shows the last 10 runs, the 30-day count and the count since Jan 2024 (OUT008 chilled: 5 in 30 days, 151 of 440) |
| Swap suggestion | The best swap is marked "Suggested"; consequences (served, deferred, chilled m³, trip minutes, violations) show before applying |
| Rule check | A plan that breaks a rule cannot be published |
| Store notice | Every waiting store is told the reason and the new date when the plan is published |
| Realistic time | Expected times use real travel (1.34× the plan); a trip that passes the rules but will be late is shown **at risk** (VEH006 Trip 2, ~08:25) |
| Live | A progress timeline per vehicle, not a GPS map |

### Loading, delivery and receipt
| Decision | Detail |
|---|---|
| Load order | Load list in reverse stop order (first stop loaded last), with a cargo-space diagram |
| Shortfall | The loader flags missing or damaged cases **before departure** (reason, count, photo); the dispatcher, driver and store are told |
| Proof of delivery | Cases handed over, photo and signature, recorded offline if needed |
| Three counts | Loaded (loader) → handed over (driver) → received (store); the store sees all three when confirming |
| Problems | Structured reports with reason codes, not chat |

### Degradation scenario
Driver offline on the Kandy corridor (real route R025028, VEH043). The phone keeps working and queues records; they sync later with unique IDs, so nothing is counted twice. The dispatcher sees the driver as **Offline** with the last contact time, not as Late. If the plan changed while the driver was offline, what happened on the ground wins and the dispatcher gets a conflict to review.

### Left out on purpose (Figma page 04)
Account and user management, driver–vehicle assignment, forecast screen, live GPS map, chat, Sinhala/Tamil, native apps, invoices and credit notes, product-level order lines, profile and settings, turn-by-turn navigation, editing orders after the 4 PM cutoff.

### Visual design (style guide v4)
| Decision | Detail |
|---|---|
| Colour | Blue #2463EB is the action colour: solid only on the one main action of a screen, tonal on the item being worked on now. Status colours (on time, at risk, deferred, late, offline, queued) and the chilled marker are used only for status. Brands get no colour |
| Tokens | 36 colour tokens, Light and Dark, same names; screens use tokens only |
| Type | IBM Plex Sans for the interface, IBM Plex Mono for IDs, times and quantities; sizes 12 / 14 / 16 / 20 / 24 / 32; screen titles 32 Bold |
| Shape | Cards radius 16, controls 12, buttons, chips and badges fully rounded; cards have no border; soft shadow in light mode only |
| Icons | Lucide only, one meaning per icon, no emojis |
| Targets | At least 44 px on phones, 48 px driver main actions, 56 px loader buttons with 12 px gaps |
| Contrast | WCAG 2.2 AA in both themes; status is always colour + icon + label |

### Design data
All screens use the S1 peak day and real records: 85 orders from 59 outlets, 28 available vehicles. The story follows OUT008: its chilled order is deferred by auto-plan (72 served, 13 deferred, 93.7 of 181.6 m³ chilled), then the dispatcher swaps VEH006 Trip 1 from Kurunegala to Colombo (73 served, 12 deferred, 101.5 m³; OUT066 and OUT067 wait instead). The plan passed the official `check_allocation.py`.
