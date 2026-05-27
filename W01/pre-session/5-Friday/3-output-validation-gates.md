---
week: W01
day: Fri
topic_slug: output-validation-gates
topic_title: "Output validation gates — Pydantic + Bean Validation contract"
parent_overview: W01/pre-session/5-Friday/1-DailyTopicOverview.md
estimated_minutes: 10
sources:
  - url: https://docs.spring.io/spring-framework/reference/core/validation/beanvalidation.html
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://docs.spring.io/spring-boot/reference/io/validation.html
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://jakarta.ee/specifications/bean-validation/3.0/jakarta-bean-validation-spec-3.0.html
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://docs.pydantic.dev/latest/concepts/models/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://github.com/FasterXML/jackson-databind/wiki/Deserialization-Features
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
last_verified: 2026-05-26
---

# Output validation gates — Pydantic + Bean Validation contract

## 1. Learning Objectives

By the end of this reading, the learner can:

- Articulate why two-layer schema validation across a service boundary is defence-in-depth, not redundancy.
- Map Pydantic v2 constraints to Jakarta Bean Validation annotations and identify drift between them.
- Wire up `@Valid` on a Spring `@RequestBody` and surface the `MethodArgumentNotValidException` as a structured client response.
- Recognise the failure modes that single-layer validation cannot catch (transport-layer corruption, deserialisation gaps, semantic drift).
- Use schema-drift detection — typically a contract test — to keep the two layers consistent over time.

## 2. Introduction

When two services pass typed data across a network boundary, each side has its own contract enforcement. The producing service validates what it emits; the consuming service validates what it receives. Naive engineering treats this as duplication and asks which one to delete. Production engineering treats it as defence-in-depth and asks how to keep the two contracts honest with each other.

The pattern is older than LLMs — it shows up wherever a Python service feeds a Java service (or vice versa) across HTTP, in microservice meshes, in event-driven pipelines, in any system where the two languages have independent serialisation stacks. What the LLM era added is *volatility*: the upstream schema can shift because a prompt changed or a model rolled over, not because anyone deployed code. That makes the receiving-side validation gate a load-bearing safety net, not a courtesy.

This reading covers the two-layer pattern in its canonical shape — Pydantic v2 on the producing side (the AI service in Python) and Jakarta Bean Validation on the consuming side (the business service in Spring Boot). The shape generalises: anywhere two type systems meet across a wire, the same logic applies.

## 3. Core Concepts

### 3a. Why two layers?

Three failure modes motivate the double-validation pattern:

1. **Transport corruption.** A field is truncated or re-encoded en route. The producer validated the original; the consumer sees the corruption. Without consumer-side validation, the corruption flows downstream.
2. **Deserialisation gap.** The producer's JSON serialiser and the consumer's JSON deserialiser disagree about edge cases — null handling, large integers, date formats, unicode normalisation. The producer's validator never sees what the consumer parses.
3. **Schema drift.** The producer changes its schema (a prompt change, a model swap, a model-config tweak) without coordinating with the consumer. Consumer-side validation catches the drift at the boundary, not in some downstream business-logic crash three hops later.

Industry guidance from 2026 is explicit: layered validation provides defence in depth, and even when using structured-output APIs that guarantee schema compliance, application-level validation for semantic correctness remains necessary. No single layer is sufficient.

### 3b. Producer-side: Pydantic v2 strict mode

The Python producer declares a model with `extra="forbid"` and explicit constraints. It validates *before* serialising the response. If validation fails, the producer returns a structured error, not a partially-valid payload.

```python
from pydantic import BaseModel, ConfigDict, Field

class ExtractedAddress(BaseModel):
    model_config = ConfigDict(extra="forbid", frozen=True)

    street: str = Field(..., min_length=3, max_length=200)
    city: str = Field(..., min_length=2, max_length=100)
    postal_code: str = Field(..., pattern=r"^[A-Z0-9\- ]{3,12}$")
    country_iso2: str = Field(..., pattern=r"^[A-Z]{2}$")
```

### 3c. Consumer-side: Jakarta Bean Validation

The Java consumer declares a mirror class with Jakarta Bean Validation annotations and uses `@Valid` on the `@RequestBody` parameter. Spring Boot's auto-configuration wires Hibernate Validator (the reference implementation) automatically when `spring-boot-starter-validation` is on the classpath.

```java
import jakarta.validation.constraints.Pattern;
import jakarta.validation.constraints.Size;

public record ExtractedAddress(
    @Size(min = 3, max = 200) String street,
    @Size(min = 2, max = 100) String city,
    @Pattern(regexp = "^[A-Z0-9\\- ]{3,12}$") String postalCode,
    @Pattern(regexp = "^[A-Z]{2}$") String countryIso2
) {}
```

The controller:

```java
import jakarta.validation.Valid;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;

@PostMapping("/addresses")
public ResponseEntity<AddressId> ingest(@Valid @RequestBody ExtractedAddress request) {
    // request is guaranteed to satisfy every annotation above.
    return ResponseEntity.ok(addressService.persist(request));
}
```

When validation fails, Spring throws `MethodArgumentNotValidException`, which a `@ControllerAdvice` global handler converts into a structured `400 Bad Request` response naming the offending fields.

### 3d. The mapping table — keep it deliberate

Every Pydantic constraint has a Bean Validation cousin. The mapping is not 1:1 in every case, and keeping the two definitions consistent requires discipline.

| Pydantic v2 | Bean Validation (Jakarta) |
|---|---|
| `Field(..., min_length=N, max_length=M)` on `str` | `@Size(min = N, max = M)` |
| `Field(..., ge=N, le=M)` on numbers | `@Min(N) @Max(M)` |
| `Field(..., gt=0)` | `@Positive` |
| `Field(..., pattern=r"...")` | `@Pattern(regexp = "...")` |
| `Field(...)` (required) | `@NotNull` |
| `Field(..., min_length=1)` on `str` | `@NotBlank` (also rejects whitespace-only) |
| `model_config = ConfigDict(extra="forbid")` | Jackson `@JsonIgnoreProperties(ignoreUnknown = false)` *or* `spring.jackson.deserialization.fail-on-unknown-properties=true` |

Two cells need a callout. First, `@NotBlank` is stricter than `min_length=1` because it also rejects pure whitespace; if your Pydantic side does not strip whitespace, the mismatch will fire on legitimate-looking inputs. Second, `extra="forbid"` has no native Bean Validation equivalent — you control it at the Jackson layer, either per-class or globally via configuration.

### 3e. Contract testing — the third leg

Two layers stay in sync because something tests both. The minimum-viable approach is a contract test that ships in CI:

1. Round-trip a representative valid payload through Pydantic + JSON + Jackson + Bean Validation. Both layers accept.
2. Round-trip a payload that violates one constraint (e.g., too-short `street`). Both layers reject, and they reject on the *same* field.
3. Round-trip a payload with an extra field. Producer rejects (extra="forbid"); consumer rejects (Jackson fail-on-unknown).

Without this test, drift is silent until production. With it, drift fails CI on the PR that introduced it.

## 4. Generic Implementation

A worked example outside federal acquisitions: a logistics platform passes parsed shipping-label data from a Python OCR service to a Java order-management service.

**Python producer (Pydantic):**

```python
from pydantic import BaseModel, ConfigDict, Field

class ShippingLabel(BaseModel):
    model_config = ConfigDict(extra="forbid", frozen=True)

    tracking_number: str = Field(..., pattern=r"^[A-Z0-9]{10,30}$")
    weight_grams: int = Field(..., gt=0, le=70_000)
    destination_postcode: str = Field(..., min_length=3, max_length=12)


# Endpoint: returns validated JSON or raises ValidationError
@app.post("/extract-label")
def extract(image: bytes) -> ShippingLabel:
    raw = ocr_then_llm_extract(image)
    return ShippingLabel.model_validate(raw)   # strict mode; failures propagate
```

**Java consumer (Bean Validation):**

```java
public record ShippingLabel(
    @Pattern(regexp = "^[A-Z0-9]{10,30}$") String trackingNumber,
    @Positive @Max(70_000) int weightGrams,
    @Size(min = 3, max = 12) String destinationPostcode
) {}

@RestController
class IngestController {
    @PostMapping("/ingest-label")
    public ResponseEntity<Void> ingest(@Valid @RequestBody ShippingLabel label) {
        orderService.attachLabel(label);
        return ResponseEntity.accepted().build();
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ValidationError> onValidationError(
            MethodArgumentNotValidException ex) {
        var errors = ex.getBindingResult().getFieldErrors().stream()
            .map(fe -> new FieldError(fe.getField(), fe.getDefaultMessage()))
            .toList();
        return ResponseEntity.badRequest().body(new ValidationError(errors));
    }
}
```

The two definitions are *intentionally redundant* — they encode the same contract twice in two type systems, with a contract test in CI to keep them honest.

## 5. Real-world Patterns

**E-commerce — checkout payload across a polyglot stack:** A retail platform runs Python services for recommendations and a Java service for the checkout itself. The recommendation service emits a `CartLineItems` payload validated by Pydantic; the checkout service validates incoming `@RequestBody` data with Bean Validation. When a recommendation-service prompt change once dropped the `quantity` field for premium subscribers, the checkout service rejected the payload with a clean `MethodArgumentNotValidException` instead of silently quoting wrong totals. The two-layer gate moved an invisible-billing bug into a loud 400-response visible on dashboards.

**Healthcare — lab-result ingestion:** A clinical data platform receives lab results from a Python parser that extracts structured fields from PDF reports. The downstream Spring-based clinical workflow service uses Bean Validation with custom `@Constraint` annotations for LOINC-code conformance. The team treats producer + consumer as adversarial — each assumes the other is wrong — and contract tests run nightly against synthetic edge cases.

**Gaming — match-making event validation:** A live-service game emits match events from a Python event-shaping service; a Java match-history service consumes them. Bean Validation on the consumer side has caught at least three production incidents where transport corruption mangled UTF-8 in player handles — corruption that the producer's Pydantic gate never saw because the bytes were intact at emission time.

**Fintech — KYC document extraction:** A banking-compliance pipeline extracts identity-document fields in Python, sends them to a Java compliance engine that runs the watchlist match. The two layers carry identical regex constraints for passport numbers; a quarterly drift-detection job diffs the regex strings and alerts compliance engineering on divergence. The discipline is the test, not the code.

## 6. Best Practices

- Treat the producer and consumer as adversarial — assume the other side is wrong, and validate accordingly.
- Keep the constraint definitions in a *table* (or generated schema) that both sides reference; do not let them drift independently.
- Configure Jackson to fail on unknown properties globally (`spring.jackson.deserialization.fail-on-unknown-properties=true`) — the consumer's equivalent of `extra="forbid"`.
- Surface `MethodArgumentNotValidException` as a structured `400` response with field-level error detail; never as a generic stack trace.
- Ship a contract test in CI that round-trips representative valid and invalid payloads through both layers.
- Log validation failures on the consumer side with the upstream request id so producer-side debugging is one query away.
- Version the shared schema (semver) so prompt-driven schema changes need an explicit, reviewable bump.

## 7. Hands-on Exercise

**Task (10–15 min):** Define a `BookOrder` contract on both sides:

- Python (Pydantic v2): `isbn` (regex `^\d{13}$`), `quantity` (1..100), `gift_message` (optional, max 280 chars), `model_config = ConfigDict(extra="forbid")`.
- Java (Bean Validation): mirror record with `@Pattern`, `@Min`/`@Max`, `@Size`, and global Jackson config `fail-on-unknown-properties=true`.

Write a short script that POSTs three payloads to a Spring controller and asserts:

1. A valid payload returns `200`.
2. A payload with `quantity = 0` returns `400` with the error pointing at `quantity`.
3. A payload with an extra `wrap_color` field returns `400` referencing the unknown field.

**What good looks like:** The two definitions encode the same constraints, named the same way, with bounds that match. The error responses name the exact violating field. The test passes only when *both* layers agree on each case — if the Pydantic side accepts something the Java side rejects (or vice versa), the test fails until you reconcile.

## 8. Key Takeaways

- *Why validate twice?* Because transport corruption, deserialisation gaps, and upstream schema drift each defeat single-layer validation.
- *How do I keep the two layers consistent?* A contract test in CI that exercises matched accept/reject cases on both sides.
- *What's the Bean Validation analogue of `extra="forbid"`?* Jackson's `fail-on-unknown-properties`, configured globally or per-class.
- *How do I surface validation errors to clients?* A `@ControllerAdvice` handler for `MethodArgumentNotValidException` that returns a structured `400` with field-level detail.
- *Where does the validation discipline live?* In the table that maps Pydantic constraints to Bean Validation annotations — kept deliberate, reviewed on every schema change.

## Sources

1. [Java Bean Validation (Spring Framework docs)](https://docs.spring.io/spring-framework/reference/core/validation/beanvalidation.html) — retrieved 2026-05-26
2. [Validation (Spring Boot docs)](https://docs.spring.io/spring-boot/reference/io/validation.html) — retrieved 2026-05-26
3. [Jakarta Bean Validation 3.0 Specification](https://jakarta.ee/specifications/bean-validation/3.0/jakarta-bean-validation-spec-3.0.html) — retrieved 2026-05-26
4. [Pydantic Models (Pydantic docs)](https://docs.pydantic.dev/latest/concepts/models/) — retrieved 2026-05-26
5. [Jackson Deserialization Features (FasterXML jackson-databind wiki)](https://github.com/FasterXML/jackson-databind/wiki/Deserialization-Features) — retrieved 2026-05-26

Last verified: 2026-05-26
