RewriteLocationResponseHeaderToOrigin.kt

```kotlin
@Component
class RewriteLocationResponseHeadersToOrigin: GlobalFilter {
 override fun filter(exchange: ServerWebExchange, chain: GatewayFilterChain): Mono<Void> {
     return chain.filter(exchange).then(
         Mono.fromRunnable {
             val response = exchange.response
             val proxiedLocationResponse = exchange.response.headers.location
             if (proxiedLocationResponse != null) {
                 val originLocationResponse = rewriteLocationToOrigin(proxiedLocationResponse, exchange.request)
                 response.headers.set(HttpHeaderNames.LOCATION.toString(), originLocationResponse.toString())
             }
             exchange.mutate().response(response).build()
         }
     )
 } 
 
 fun rewriteLocationToOrigin(location: URI, originRequest: ServerHttpRequest): URI {
     return UriComponentsBuilder.fromUri(location)
         .scheme(originRequest.uri.scheme)
         .host(originRequest.uri.host)
         .port(originRequest.uri.port)
         .build()
         .toUri()
 }  
}
```

RewriteLocationResponseHeaderToOriginTest.kt

```kotlin
internal class RewriteLocationResponseHeaderToOriginTest {
    private val slot = slot<ServerWebExchange>()
    private val filterChain: GatwayFilterChain = mockk()
    private val filter = RewriteLocationResponseHeaderToOrigin()
    
    @BeforeEach
    fun setup() {
        every { filterChain.filter(capture(slot)) } returns Mono.empty() 
    }
    
    @Test
    fun `replace a location response header with the origin host`() {
        val originUrl = "http://external-domain.ch" 
        val locationResponse = "http://internal-domain:9001"
        
        makeRequest(originUrl, locationResponse)
        val actualLocation = slot.captured.response.headers.location?.toString()
        assertThat(actualLocation).isEqualTo("http://external-domain.ch")
    }
    
    private fun makeRequest(originUrl: String, locationResponse: String) {
        val request = MockServerHttpRequest.get(originUrl).build()
        val exchange = MockServerWebExchange.from(request)
        exchange.reponse.header.location = URI(locationResponse)
        filter.filter(exchange, filterChain).block()
    }
}
```
