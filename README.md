Part 1: Code Review & Debugging
Issues I Identified
While going through the given API code, I noticed a few problems:
First, there is no input validation. The code directly accesses fields like name, sku, etc. If any field is missing, the API will crash.
Second, there is no check for SKU uniqueness. Since SKU should be unique, this can lead to duplicate products in the system.
Third, the code is using two separate database commits. This is risky because if the product is created but inventory creation fails, the system will have inconsistent data.
Also, the data modeling is not correct. The product is directly linked to a warehouse, but according to the requirements, a product can exist in multiple warehouses.
Another issue is no error handling. If something fails in the database, the system will not handle it properly.
Lastly, price is not validated properly. It should be stored as a decimal and checked before saving.

Impact in Real-World Usage
These issues can cause serious problems in production:
Missing fields can crash the API.
Duplicate SKUs can create confusion in inventory tracking.
Partial data (product without inventory) can break business logic.
Wrong database design will make scaling difficult.
Lack of error handling can make debugging very hard. 
Improved Version of the Code


@app.route('/api/products', methods=['POST'])
def create_product():
    data = request.json

    if not data or 'name' not in data or 'sku' not in data:
        return {"error": "Missing required fields"}, 400

    if Product.query.filter_by(sku=data['sku']).first():
        return {"error": "SKU already exists"}, 400

    try:
        product = Product(
            name=data['name'],
            sku=data['sku'],
            price=float(data.get('price', 0))
        )

        db.session.add(product)
        db.session.flush()

        inventory = Inventory(
            product_id=product.id,
            warehouse_id=data['warehouse_id'],
            quantity=data.get('initial_quantity', 0)
        )

        db.session.add(inventory)
        db.session.commit()

        return {"message": "Product created", "product_id": product.id}, 201

    except Exception as e:
        db.session.rollback()
        return {"error": str(e)}, 500

        

Part 2: Database Design

Proposed Schema
To support the requirements, I designed the following tables:
Companies: Stores company details
Warehouses: Each company can have multiple warehouses
Products: Stores product details (SKU is unique)
Inventory: Tracks product quantity in each warehouse
Inventory Logs: Keeps history of stock changes
Suppliers: Stores supplier information
Product_Suppliers: Maps products to suppliers
Bundles: Handles products that are combinations of other products

Questions I Would Ask
Since some requirements are incomplete, I would clarify:
Can one product have multiple suppliers?
What exactly counts as “recent sales activity”?
Should inventory updates happen in real time?
Can bundles contain other bundles?
Do we need to track historical inventory per warehouse?

Design Decisions
I made SKU unique to avoid duplicates.
I separated Inventory from Product so one product can exist in multiple warehouses.
I added an Inventory Logs table to track changes over time.
I used a many-to-many relationship for suppliers to keep it flexible.


Part 3: API Implementation
Approach
To build the low-stock alert system:
First, get all warehouses of the company
Then fetch inventory data for each warehouse
Check if stock is below threshold
Filter only products with recent sales
Attach supplier details for reordering

Implementation (Spring Boot)
Since I am more comfortable with Java, I implemented this using Spring Boot.
@RestController
@RequestMapping("/api/companies")
public class AlertController {

   @GetMapping("/{companyId}/alerts/low-stock")
   public ResponseEntity<?> getLowStockAlerts(@PathVariable Long companyId) {

       List<AlertResponse> alerts = alertService.getLowStockAlerts(companyId);

       return ResponseEntity.ok(Map.of(
               "alerts", alerts,
               "total_alerts", alerts.size()
       ));
   }
}

Logic Explanation
Loop through all warehouses of the company
For each warehouse, check inventory
Compare quantity with threshold
Skip products without recent sales
Add supplier info in response

Edge Cases Considered
Company has no warehouses
Product has no recent sales
Supplier information is missing
Large data (can be optimized using pagination and joins)

Assumptions
Each product has a defined low-stock threshold
Sales data is available to check recent activity
Each product has at least one supplier 



