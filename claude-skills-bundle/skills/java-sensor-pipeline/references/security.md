# Security

Two surfaces in this project. The pipeline runner side has almost nothing; the Orchestration Manager side has the full edge.

## Pipeline runner → Orchestration Manager

**No authentication.** Same JVM, same deployable, internal `Sinks.Many` hand-off. Adding auth here adds no security and costs latency. Don't.

## Client → Orchestration Manager REST

Spring Security reactive chain, OAuth2 resource server validating JWTs issued by an external identity provider.

```java
@Configuration
@EnableWebFluxSecurity
public class OrchestrationSecurityConfig {

    @Bean
    SecurityWebFilterChain chain(ServerHttpSecurity http) {
        return http
            .csrf(ServerHttpSecurity.CsrfSpec::disable)
            .authorizeExchange(ex -> ex
                .pathMatchers("/actuator/health").permitAll()
                .pathMatchers("/api/**").authenticated()
                .anyExchange().denyAll())
            .oauth2ResourceServer(o -> o.jwt(jwt -> jwt
                .jwtAuthenticationConverter(grantedAuthoritiesExtractor())))
            .build();
    }

    private Converter<Jwt, Mono<AbstractAuthenticationToken>> grantedAuthoritiesExtractor() {
        JwtGrantedAuthoritiesConverter c = new JwtGrantedAuthoritiesConverter();
        c.setAuthoritiesClaimName("roles");
        c.setAuthorityPrefix("ROLE_");
        JwtAuthenticationConverter j = new JwtAuthenticationConverter();
        j.setJwtGrantedAuthoritiesConverter(c);
        return new ReactiveJwtAuthenticationConverterAdapter(j);
    }
}
```

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: ${OAUTH_ISSUER}
          jwk-set-uri: ${OAUTH_JWKS}
```

CSRF disabled because this is a stateless JSON API with token auth. If you add form endpoints, re-enable for those paths only.

## Input validation inside the pipeline

Two layers: frame-level (stage 2) and value-level (stage 3).

**Stage 2 — frame validation:**
- Checksum mismatch → drop, increment `malformed_frames_total{sensor=...}`.
- Length mismatch, bad magic bytes → drop, same metric.
- Never throw. A throw terminates the `Flux` and kills the pipeline.

**Stage 3 — value validation:**
- Out-of-physical-range values (e.g. temperature sensor reporting 10000°C) → drop, increment `implausible_readings_total{sensor=...}`.
- NaN/Infinity → drop, same metric.
- Future timestamps beyond clock-skew tolerance → clamp to now, log at DEBUG.

**What stage 3 does not do:**
- Rate-limiting — that's the transport's job if needed.
- Content-based authorization — sensors don't have identity beyond what's in their config.

## REST input validation (Orchestration Manager)

Use `jakarta.validation` annotations on request DTOs. WebFlux honors them when the DTO is annotated `@Validated`.

```java
public record UpdateCalibrationRequest(
    @NotBlank String sensorId,
    @NotNull Map<String, @NotNull Double> params
) {}

@RestController
@RequestMapping("/api/sensors")
@Validated
public class SensorController {

    @PutMapping("/{id}/calibration")
    public Mono<Void> updateCalibration(
            @PathVariable String id,
            @Valid @RequestBody UpdateCalibrationRequest req) {
        if (!id.equals(req.sensorId())) {
            return Mono.error(new ResponseStatusException(HttpStatus.BAD_REQUEST, "id mismatch"));
        }
        return service.updateCalibration(req);
    }
}
```

**Don't trust any field.** Validate explicitly. `@Valid` is nice but it's not a substitute for thinking about what an attacker controls.

## Secrets

No secrets in YAML. No secrets in SQLite. No secrets in `application.properties` committed to git.

- OAuth issuer URLs and JWK URIs: environment variables.
- Any future DB credentials: environment variables or a secrets mount.
- Sensor API keys (if a sensor protocol needs one): SQLite, but in a `sensor_secret` table — never in `sensor` or `sensor_protocol_config`.
