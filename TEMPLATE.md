# <Design name>

## 1. Requirements
**Functional** (by phase) · **Non-functional** (scale, availability, latency, consistency) · **Out of scope**

## 2. Back-of-the-envelope — traffic & scale
Throughput (avg/peak) · total records · key/ID size

## 3. API
Endpoint · request · success code · failure codes

## 4. Data model
Tables · keys · indexes · why this DB

## 5. Back-of-the-envelope — storage
Row size (from the data model) · total data · index size · what fits in RAM · disk

## 6. High-level diagram
`diagram.png` (source: `diagram.excalidraw`)

## 7. Deep dives
The 2–3 parts that matter most for this problem

## 8. NFR walk-back
Each non-functional requirement → the part of the design that meets it

## 9. Doc fixes
What's wrong or missing in this file (checkboxes)

## 10. Self-check
Questions I must answer out loud without looking (tick when I can)
