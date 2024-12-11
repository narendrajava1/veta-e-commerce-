# veta-e-commerce-

Let’s break down the implementation for each of the requested features in a Spring Boot + React e-commerce project. I’ll go through them step by step:

1. User Authentication and JWT Integration

Backend: Spring Boot

	1.	Add Dependencies:
Add these to your pom.xml:

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt</artifactId>
    <version>0.9.1</version>
</dependency>


	2.	Create User Entity:

@Entity
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String username;
    private String password;
    private String role; // e.g., ROLE_USER, ROLE_ADMIN
}


	3.	Create JWT Utility:

public class JwtUtil {
    private String secret = "secretKey";

    public String generateToken(String username) {
        return Jwts.builder()
            .setSubject(username)
            .setIssuedAt(new Date())
            .setExpiration(new Date(System.currentTimeMillis() + 1000 * 60 * 60 * 10))
            .signWith(SignatureAlgorithm.HS256, secret)
            .compact();
    }

    public String extractUsername(String token) {
        return Jwts.parser().setSigningKey(secret).parseClaimsJws(token).getBody().getSubject();
    }
}


	4.	Authentication Filter:
Create a JwtRequestFilter to intercept requests and validate tokens.
	5.	Security Configuration:

@Configuration
@EnableWebSecurity
public class SecurityConfig extends WebSecurityConfigurerAdapter {
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.csrf().disable()
            .authorizeRequests()
            .antMatchers("/api/auth/**").permitAll()
            .anyRequest().authenticated();
    }
}


	6.	Auth Controller:

@RestController
@RequestMapping("/api/auth")
public class AuthController {
    @Autowired
    private AuthenticationManager authenticationManager;

    @Autowired
    private JwtUtil jwtUtil;

    @PostMapping("/login")
    public ResponseEntity<?> login(@RequestBody AuthRequest authRequest) {
        authenticationManager.authenticate(
            new UsernamePasswordAuthenticationToken(authRequest.getUsername(), authRequest.getPassword())
        );
        String token = jwtUtil.generateToken(authRequest.getUsername());
        return ResponseEntity.ok(new AuthResponse(token));
    }
}

Frontend: React

	1.	Axios Interceptor for JWT:

import axios from "axios";

axios.interceptors.request.use((config) => {
    const token = localStorage.getItem("token");
    if (token) {
        config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
});


	2.	Login Component:

import React, { useState } from "react";
import axios from "axios";

const Login = () => {
    const [username, setUsername] = useState("");
    const [password, setPassword] = useState("");

    const handleSubmit = async (e) => {
        e.preventDefault();
        const response = await axios.post("http://localhost:8080/api/auth/login", { username, password });
        localStorage.setItem("token", response.data.token);
    };

    return (
        <form onSubmit={handleSubmit}>
            <input type="text" onChange={(e) => setUsername(e.target.value)} placeholder="Username" />
            <input type="password" onChange={(e) => setPassword(e.target.value)} placeholder="Password" />
            <button type="submit">Login</button>
        </form>
    );
};

export default Login;

2. Admin Panel for Managing Products and Orders

Backend: Add Product and Order Endpoints

	1.	Add POST, PUT, DELETE endpoints in ProductController and OrderController.
	2.	Secure them with @PreAuthorize("hasRole('ADMIN')").

Frontend:

	1.	Use React Router for an admin route:

<Route path="/admin" element={<AdminPanel />} />


	2.	Create an AdminPanel component for CRUD operations:
	•	Fetch products and orders from APIs.
	•	Add forms for adding/editing products.

3. Shopping Cart Logic and Integration

Frontend:

	1.	Cart Context:

import { createContext, useContext, useReducer } from "react";

const CartContext = createContext();

const cartReducer = (state, action) => {
    switch (action.type) {
        case "ADD_TO_CART":
            return [...state, action.payload];
        case "REMOVE_FROM_CART":
            return state.filter((item) => item.id !== action.payload.id);
        default:
            return state;
    }
};

export const CartProvider = ({ children }) => {
    const [cart, dispatch] = useReducer(cartReducer, []);
    return <CartContext.Provider value={{ cart, dispatch }}>{children}</CartContext.Provider>;
};

export const useCart = () => useContext(CartContext);


	2.	Cart Page:

import { useCart } from "../context/CartContext";

const CartPage = () => {
    const { cart, dispatch } = useCart();

    return (
        <div>
            {cart.map((item) => (
                <div key={item.id}>
                    <h2>{item.name}</h2>
                    <button onClick={() => dispatch({ type: "REMOVE_FROM_CART", payload: item })}>
                        Remove
                    </button>
                </div>
            ))}
        </div>
    );
};

4. Payment Gateway Integration (Stripe)

Backend:

	1.	Add Stripe dependency:

<dependency>
    <groupId>com.stripe</groupId>
    <artifactId>stripe-java</artifactId>
    <version>20.124.0</version>
</dependency>


	2.	Create a PaymentController:

@RestController
@RequestMapping("/api/payment")
public class PaymentController {
    @PostMapping
    public ResponseEntity<String> processPayment(@RequestBody PaymentRequest request) {
        Stripe.apiKey = "your-stripe-secret-key";
        Charge charge = Charge.create(Map.of(
            "amount", request.getAmount(),
            "currency", "usd",
            "source", request.getToken()
        ));
        return ResponseEntity.ok("Payment successful!");
    }
}



Frontend:

	1.	Install react-stripe-checkout:

npm install react-stripe-checkout


	2.	Create a CheckoutButton:

import StripeCheckout from "react-stripe-checkout";
import axios from "axios";

const CheckoutButton = ({ amount }) => {
    const handleToken = (token) => {
        axios.post("http://localhost:8080/api/payment", { token, amount });
    };

    return <StripeCheckout token={handleToken} stripeKey="your-public-key" amount={amount * 100} />;
};

export default CheckoutButton;

5. Order Management System

Backend:

	1.	Add an Order entity:

@Entity
public class Order {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private Long userId;
    private LocalDateTime date;
}


	2.	Add an endpoint for viewing user orders.

Frontend:

	•	Show past orders in a MyOrders component using the /orders API.

6. Deploying the App using Docker

	1.	Backend Dockerfile:

FROM openjdk:17
COPY target/ecommerce.jar ecommerce.jar
ENTRYPOINT ["java", "-jar", "/ecommerce.jar"]


	2.	Frontend Dockerfile:

FROM node:16
WORKDIR /app
COPY . .
RUN npm install
RUN npm run build
EXPOSE 3000
CMD ["npx", "serve", "-s", "build"]


	3.	Docker Compose:

version: "3"
services:
  backend:
    build: ./backend
    ports:
      - "8080:8080"
  frontend:
    build: ./frontend
    ports:
      - "3000:3000"



Would you like more specific code for any part of the project?
