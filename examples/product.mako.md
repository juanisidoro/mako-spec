---
mako: "1.0"
type: product
entity: "Nike Air Max 90"
updated: 2026-02-13
tokens: 245
language: en
summary: "Mid-range casual running shoe by Nike, 79.99 EUR"

actions:
  - name: add_to_cart
    description: "Add this product to the shopping cart"
    endpoint: /api/cart/add
    method: POST
    params:
      - name: size
        type: string
        required: true
        description: "Shoe size (EU sizing, e.g., '42')"
      - name: quantity
        type: integer
        required: false
        description: "Number of pairs (default: 1)"

  - name: check_availability
    description: "Check stock availability by size"
    endpoint: /api/stock/check
    method: GET
    params:
      - name: size
        type: string
        required: true
        description: "Size to check"

links:
  internal:
    - url: /category/running-shoes
      context: "Browse all running shoes"
      type: parent
    - url: /product/nike-pegasus-40
      context: "Lighter alternative, 20 EUR more"
      type: sibling
    - url: /guides/choosing-running-shoes
      context: "Guide to picking the right running shoe"
      type: reference
  external:
    - url: https://nike.com/air-max-90
      context: "Official Nike product page"
      type: source

related:
  - /product/adidas-ultraboost
  - /product/new-balance-1080
  - /product/asics-gel-nimbus

tags:
  - running
  - shoes
  - nike
  - casual
---

# Nike Air Max 90

Mid-range casual running shoe by Nike. Popular for daily wear and light running.

## Key Facts
- Price: 79.99 EUR (was 149.99 EUR, 47% off)
- Availability: In stock
- Sizes: EU 38-46
- Material: Leather upper, mesh panels, Air Max cushioning
- Weight: 340g (size 42)
- Rating: 4.3/5 (234 reviews)

## Context
Positioned as an affordable entry into Nike's Air Max line. Direct competitors at similar price: Adidas Runfalcon (69€), Puma Velocity (74€). Premium alternatives: Adidas Ultraboost (129€), New Balance 1080 (139€).

Best for: casual running, daily commute, gym. Not recommended for: trail running, marathon training.

## Reviews Summary
Positive: comfortable all-day wear, classic design, good value at sale price.
Negative: narrow fit (size up if wide feet), outsole wears fast on pavement, limited color options in stock.
