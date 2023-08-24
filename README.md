# Customer-Application
//Controller
@RestController
@RequestMapping("/api/products")
public class ProductController {

    @Autowired
    private ProductService productService;

    @GetMapping
    public ResponseEntity<List<ProductDTO>> getAllAvailableProducts(@RequestParam String category,
                                                                    @RequestParam @DateTimeFormat(iso = DateTimeFormat.ISO.DATE_TIME) LocalDateTime startTime,
                                                                    @RequestParam @DateTimeFormat(iso = DateTimeFormat.ISO.DATE_TIME) LocalDateTime endTime) {
        List<ProductDTO> availableProducts = productService.getAvailableProducts(category, startTime, endTime);
        return ResponseEntity.ok(availableProducts);
    }
}

//Service
public class ProductService {

    @Autowired
    private ProductRepository productRepository;

    public List<ProductDTO> getAvailableProducts(String category, LocalDateTime startTime, LocalDateTime endTime) {
        List<Product> availableProducts = productRepository.findAvailableProducts(category, startTime, endTime);
        // Convert entities to DTOs and calculate costs based on duration
        List<ProductDTO> productDTOs = availableProducts.stream()
                .map(product -> convertToDTOWithCost(product, startTime, endTime))
                .collect(Collectors.toList());
        return productDTOs;
    }
    
    // Implement the convertToDTOWithCost method to calculate the cost based on the duration and product's pricing strategy.
}

//Repository
public interface ProductRepository extends JpaRepository<Product, Long> {

    @Query("SELECT p FROM Product p WHERE p.category = :category AND NOT EXISTS " +
           "(SELECT b FROM Booking b WHERE b.product = p AND :startTime < b.endTime AND :endTime > b.startTime)")
    List<Product> findAvailableProducts(@Param("category") String category,
                                        @Param("startTime") LocalDateTime startTime,
                                        @Param("endTime") LocalDateTime endTime);
}

//Create Rental Booking
//Controller
@RestController
@RequestMapping("/api/bookings")
public class BookingController {

    @Autowired
    private BookingService bookingService;

    @PostMapping
    public ResponseEntity<String> createBooking(@RequestBody BookingRequest bookingRequest) {
        if (bookingService.isProductAvailable(bookingRequest.getProductId(), bookingRequest.getStartTime(), bookingRequest.getEndTime())) {
            bookingService.createBooking(bookingRequest);
            return ResponseEntity.ok("Booking created successfully");
        } else {
            return ResponseEntity.status(HttpStatus.BAD_REQUEST).body("Product is not available for the selected duration");
        }
    }
}

//Service
public class BookingService {

    @Autowired
    private BookingRepository bookingRepository;

    public boolean isProductAvailable(Long productId, LocalDateTime startTime, LocalDateTime endTime) {
        return bookingRepository.countOverlappingBookings(productId, startTime, endTime) == 0;
    }

    public void createBooking(BookingRequest bookingRequest) {
        Booking booking = new Booking();
        booking.setProductId(bookingRequest.getProductId());
        booking.setStartTime(bookingRequest.getStartTime());
        booking.setEndTime(bookingRequest.getEndTime());
        // Set other booking details
        
        bookingRepository.save(booking);
    }
}

//Repository
public interface BookingRepository extends JpaRepository<Booking, Long> {

    @Query("SELECT COUNT(b) FROM Booking b WHERE b.productId = :productId AND :startTime < b.endTime AND :endTime > b.startTime")
    long countOverlappingBookings(@Param("productId") Long productId,
                                  @Param("startTime") LocalDateTime startTime,
                                  @Param("endTime") LocalDateTime endTime);
}


