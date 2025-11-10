# Bynry Backend Engineering Intern - Case Study
### Author: Yash Khapre

## Part 1: Code Review & Fix

### Problems Found in the Provided Code
1. No Input Validation 
    Incoming request data (such as missing fields or invalid types) is not validated by the code.  

    Inaccurate database entries or runtime issues may result from this.

 2. There is no error handling  
    Try-catch (in Java) and try-except (in Python) blocks are not present.  
    The API will crash rather than provide a suitable error answer if a database query fails.

 3. No database integration and hard-coded data  
    The information is kept in an in-memory object or list.  
    This is impractical for production because all data is lost when the server restarts.

4.  No Management of Unique Product IDs  
 Product IDs are neither automatically created nor verified to be unique.  
    Duplicate IDs or mismatched data could result from this.

 5.  The structure of the REST API is incorrect.  
 Certain endpoints do not adhere to REST name rules (for example, `/addProduct` rather than `POST /products`).  
    HTTP verbs should be used correctly in RESTful APIs.

 6.  HTTP Status Codes That Are Missing  
 Frontend integration is challenging because the API does not deliver the correct status codes (`201 Created`, `404 Not Found`, etc.).

7. There is no separation of concerns  
    One file contains all of the logic (data and routing).  
    The controller, service, and model layers should ideally be kept apart.

 8. No Comments or Documentation  
    The lack of docstrings or comments outlining the functions' functions is detrimental to readability and maintainability.

---

## Here is a Corrected Version of a Code in Java

@RestController
@RequestMapping("/api/products")
public class ProductController {

    @Autowired
    private ProductRepository productRepository;

    @Autowired
    private InventoryRepository inventoryRepository;

    @PostMapping
    public ResponseEntity<?> createProduct(@RequestBody ProductRequest request) {

        //  1. Validate input
        if (request.getName() == null || request.getSku() == null || request.getPrice() == null) {
            return ResponseEntity.badRequest().body("Missing required fields");
        }

        //  2. Check SKU uniqueness
        if (productRepository.existsBySku(request.getSku())) {
            return ResponseEntity.status(HttpStatus.CONFLICT).body("SKU already exists");
        }

        try {
            //  3. Save product
            Product product = new Product();
            product.setName(request.getName());
            product.setSku(request.getSku());
            product.setPrice(request.getPrice());
            productRepository.save(product);

            //  4. Add initial inventory entry
            Inventory inventory = new Inventory();
            inventory.setProduct(product);
            inventory.setWarehouseId(request.getWarehouseId());
            inventory.setQuantity(request.getInitialQuantity() != null ? request.getInitialQuantity() : 0);
            inventoryRepository.save(inventory);

            return ResponseEntity.status(HttpStatus.CREATED)
                    .body(Map.of("message", "Product created successfully", "productId", product.getId()));

        } catch (Exception e) {
            //  5. Handle errors
            return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
                    .body("Error creating product: " + e.getMessage());
        }
    }
}


## Part 2: Database Design (StockFlow Schema)

###  Database Schema Design

#### Entities:
1. **Company**
   - `id` (PK)
   - `name`
   - `email`
   - `created_at`

2. **Warehouse**
   - `id` (PK)
   - `name`
   - `location`
   - `company_id` (FK → Company.id)

3. **Product**
   - `id` (PK)
   - `name`
   - `sku` (unique)
   - `price` (decimal)
   - `description` (nullable)
   - `is_bundle` (boolean)
   - `created_at`

4. **Inventory**
   - `id` (PK)
   - `product_id` (FK → Product.id)
   - `warehouse_id` (FK → Warehouse.id)
   - `quantity`
   - `last_updated`
   - (Composite Unique Key: product_id + warehouse_id)

5. **Supplier**
   - `id` (PK)
   - `name`
   - `contact_email`
   - `phone`

6. **ProductSupplier**
   - `product_id` (FK → Product.id)
   - `supplier_id` (FK → Supplier.id)
   - (Composite PK: product_id + supplier_id)

7. **InventoryHistory**
   - `id` (PK)
   - `inventory_id` (FK → Inventory.id)
   - `change_quantity`
   - `change_type` (enum: "ADD", "REMOVE", "SALE", "TRANSFER")
   - `timestamp`

---

###  Questions / Gaps to Clarify

1. What is meant by "recent sales activity"?  (For instance, the previous seven or thirty days?)  
 2. Are bundle products tracked as a mix of pre-existing SKUs or as a distinct SKU?  
 3. Should suppliers be global for all businesses or company-specific?  
 4. Does each product or warehouse have a minimum threshold?  
 5. Should low-stock alerts only show up on the dashboard or cause automated notifications (email/API)?

---

### Design Decisions

- **Indexes:**  
  Added indexes on `sku`, `warehouse_id`, and `product_id` for faster search and filtering.

- **Constraints:**  
  Enforced unique constraint on `(product_id, warehouse_id)` in `Inventory` to prevent duplicates.

- **Auditability:**  
  `InventoryHistory` maintains a complete change log for inventory movements.

- **Scalability:**  
  Using separate tables for relationships allows adding new suppliers, warehouses, and bundles easily.

---

## Part 3: API Implementation (Low-Stock Alert Endpoint)

### Endpoint
`GET /api/companies/{company_id}/alerts/low-stock`

### Example Spring Boot Implementation

```java
@RestController
@RequestMapping("/api/companies")
public class AlertController {

    @Autowired
    private AlertService alertService;

    @GetMapping("/{companyId}/alerts/low-stock")
    public ResponseEntity<Map<String, Object>> getLowStockAlerts(@PathVariable Long companyId) {
        List<LowStockAlert> alerts = alertService.getLowStockAlerts(companyId);
        Map<String, Object> response = new HashMap<>();
        response.put("alerts", alerts);
        response.put("total_alerts", alerts.size());
        return ResponseEntity.ok(response);
    }
}


##  Assumptions & Questions

1. Low-stock thresholds can be set at the product level for each product.  
 2. "Recent sales activity" refers to goods sold in the previous 30 days.  
 3. A product may be kept in more than one warehouse, but the SKU is always the same.  
 4. A `ProductSupplier` table (many-to-many) links supplier data.  
 5. The quantity of a "bundle" product is determined by the lowest stock of its individual components.  
 6. Every process, including updating inventory and creating products, is transactional and rolls back in the event of a failure.  
 7. API replies always contain `status`, `message`, and `data` and adhere to REST norms.  
 8. To ensure scalability, the database use **MySQL** with appropriate indexing.



##  Tech Stack Used

- **Language:** Java  
- **Framework:** Spring Boot  
- **Database:** MySQL  
- **Build Tool:** Maven  
- **API Design:** RESTful APIs  
- **Version Control:** GitHub  
