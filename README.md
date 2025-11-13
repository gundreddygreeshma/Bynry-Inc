Part 1: Code Review & Debugging
Task: Review and fix the product creation API.
Assumptions / Missing Requirements:
Products can exist in multiple warehouses.
SKU must be unique.
Price can be decimal.
warehouse_id and initial_quantity may be optional.


Edge Cases Considered:
Missing fields (name, sku, price).
Duplicate SKUs.
Invalid price values.
Optional fields missing.
Inventory creation fails after product saved → handled via @Transactional.


Java Spring Boot Implementation:
@RestController
@RequestMapping("/api/products")
public class ProductController {

    @Autowired
    private ProductRepository productRepository;

    @Autowired
    private InventoryRepository inventoryRepository;

    @Autowired
    private WarehouseRepository warehouseRepository;

    // Transactional ensures product + inventory creation is atomic
    @Transactional
    @PostMapping
    public ResponseEntity<?> createProduct(@RequestBody Map<String, Object> data) {
        try {
            // Validate required fields (name, sku, price)
            // Issue in original code: No validation
            // Impact: Missing field causes runtime exception, API crashes
            // Fix: Check required fields before proceeding
            if (!data.containsKey("name") || !data.containsKey("sku") || !data.containsKey("price")) {
                return ResponseEntity.badRequest().body("Missing required fields: name, sku, or price");
            }

            String name = data.get("name").toString();
            String sku = data.get("sku").toString();

            // Ensure SKU uniqueness across platform
            // Issue in original code: SKU duplicates allowed
            // Impact: Violates business rules, causes confusion in inventory tracking
            // Fix: Check if SKU exists, return conflict if duplicate
            if (productRepository.existsBySku(sku)) {
                return ResponseEntity.status(HttpStatus.CONFLICT).body("SKU already exists");
            }

            // Parse price as BigDecimal
            // Issue in original code: Price type not validated
            // Impact: Invalid decimal can crash DB or financial calculations
            // Fix: Try-catch parsing, return bad request if invalid
            BigDecimal price;
            try {
                price = new BigDecimal(data.get("price").toString());
            } catch (NumberFormatException e) {
                return ResponseEntity.badRequest().body("Invalid price format");
            }

            // Create and save product entity
            // Issue in original code: No transaction handling; product and inventory saved separately
            // Impact: If inventory creation fails, product already exists causing data inconsistency
            // Fix: Save product within @Transactional method so rollback occurs on failure
            Product product = new Product();
            product.setName(name);
            product.setSku(sku);
            product.setPrice(price);
            productRepository.save(product);

            // Handle warehouse and inventory if provided
            // Issue in original code: Assumes product exists in only one warehouse
            // Impact: Cannot track products across multiple warehouses
            // Fix: Inventory table links Product → Warehouse → Quantity
            if (data.containsKey("warehouse_id")) {
                Long warehouseId = Long.parseLong(data.get("warehouse_id").toString());

                // Check warehouse exists
                Warehouse warehouse = warehouseRepository.findById(warehouseId)
                        .orElseThrow(() -> new RuntimeException("Warehouse not found"));

                // Handle optional initial_quantity
                // Issue in original code: Assumes initial_quantity exists
                // Impact: Missing quantity crashes API
                // Fix: Default to 0 if missing
                Integer initialQuantity = 0;
                if (data.containsKey("initial_quantity")) {
                    initialQuantity = Integer.parseInt(data.get("initial_quantity").toString());
                }

                // Create inventory record for this warehouse
                // Additional context: Products can exist in multiple warehouses
                // Fix: Each warehouse gets its own inventory record for the same product
                Inventory inventory = new Inventory();
                inventory.setProduct(product);
                inventory.setWarehouse(warehouse);
                inventory.setQuantity(initialQuantity);
                inventoryRepository.save(inventory);
            }

            // Return a success response with the product ID
            Map<String, Object> response = new HashMap<>();
            response.put("message", "Product created successfully");
            response.put("product_id", product.getId());
            return ResponseEntity.ok(response);

        } catch (Exception e) {
            // Handle unexpected exceptions
            // Issue in original code: No exception handling
            // Impact: Crashes API, exposes stack traces
            // Fix: Wrap in try-catch, return Internal Server Error if something unexpected happens
            return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
                    .body("Error creating product: " + e.getMessage());
        }
    }
}

