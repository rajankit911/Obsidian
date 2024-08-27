To handle Cross-Origin Resource Sharing (CORS) in Java microservices with Spring, you can configure CORS support in the application. Spring provides the necessary components to manage CORS headers and allow or restrict cross-origin requests.

- **Enable CORS globally**: You can enable CORS for all requests by adding the **@CrossOrigin** annotation at the controller class level or by using the **WebMvcConfigurer** interface.
- **Fine-grained CORS configuration**: For more control, you can specify CORS configuration for individual endpoints or set custom headers, methods, and allowed origins.

![rest controller microservices.webp](https://images.prismic.io/turing/658bfc28531ac2845a26f359_rest_controller_microservices_8f5a32453d.webp?auto=format,compress)

- **Global CORS Configuration**: Implement a global CORS configuration bean that applies to all controllers in the application.

![cors configuration.webp](https://images.prismic.io/turing/658bfc29531ac2845a26f35a_cors_configuration_4f5fc22983.webp?auto=format,compress)

When CORS is configured properly, Java microservices can handle cross-origin requests securely and control which domains are allowed to access their endpoints.