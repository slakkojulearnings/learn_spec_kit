# Feature Specification: Loan Submission API

**Feature Branch**: `[001-loan-submission-api]`

**Created**: 2026-09-02

**Status**: Draft

**Input**: User description: "Create a REST API for submitting a loan.
The API will need the following data.
 - Customer data: full name, ID number, address (street, city, zip code), birth date, phone number, email, and current monthly income.
 - Collateral data is always a car: brand, model, manufacturing year, and license plate.
 - Proposed loan data: amount (must be a multiple of 100), tenure (between 3-48 months).

If all inputs are valid, store the data in the database as \"IN_PROGRESS\" status."

## Clarifications

### Session 2026-09-02

- Q: When a submission includes fields the API doesn't recognize (names beyond the documented customer, collateral, and loan fields), should the system reject the whole request, or silently ignore the extra fields and process the rest? → A: Reject the entire request as invalid if it contains any unrecognized field (strict schema validation).
- Q: Should monthly income and loan amount accept fractional/decimal values (e.g., 4500.50), or must they always be whole numbers? → A: Decimals allowed for both, up to 2 decimal places; the loan amount's numeric value must still reduce to a whole multiple of 100 (e.g., 300.00 is valid, 300.50 is not).
- Q: What phone number format should count as "well-formed" for validation purposes? → A: Strict E.164 format only — a leading `+`, country code, then digits, no spaces or punctuation (e.g., `+14155552671`).
- Q: Is there a minimum plausible manufacturing year for the collateral car, or should any past year up through the current year be accepted? → A: Sanity floor of 1900 only; any year 1900–current is accepted regardless of vehicle age (no business-driven "recent vehicles only" cutoff).
- Q: How should the system record whether the customer is married? → A: A `maritalStatus` enum field on Customer with values `SINGLE`, `MARRIED`, `DIVORCED`, `WIDOWED`.
- Q: Should the spouse's full name, ID number, phone number, and monthly income each follow the exact same format and validation rules as the corresponding customer fields? → A: Spouse full name, ID number, and monthly income mirror the customer's exact validation rules; the spouse's phone number uses a looser format (digits, spaces, and `+ - ( )` allowed — no strict E.164 requirement).
- Q: If the customer's marital status is not `MARRIED` but the request still includes spouse data, what should happen? → A: Accept and store the spouse data regardless of marital status; spouse data is only strictly required (and validated for completeness) when marital status is `MARRIED`.
- Q: Should the car's brand and model be validated only as non-empty free text, or must they be checked against a known reference list of valid manufacturers/models? → A: Accept any non-empty free text for both brand and model — no reference list.
- Q: Should the car's brand, model, and license plate fields have a maximum character length limit? → A: No maximum length — any non-empty string is accepted, consistent with the spec's other free-text fields.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Submit a complete, valid loan application (Priority: P1)

A caller (e.g., a loan origination front-end or partner system, acting on behalf of an applicant) submits a single request containing the applicant's personal data, the car being offered as collateral, and the proposed loan terms. The system checks that everything is present and valid, then records the application so it can move into review.

**Why this priority**: This is the core purpose of the feature — without it, no loan application can ever enter the system. It delivers the entire value of the feature on its own.

**Independent Test**: Submit a request with complete, correctly formatted customer, collateral, and loan data. Verify the response confirms acceptance with a reference identifier, and that a record exists with status "IN_PROGRESS" containing exactly the submitted data.

**Acceptance Scenarios**:

1. **Given** a request with valid customer, collateral, and loan data, **When** it is submitted, **Then** the system creates a loan application record with status "IN_PROGRESS" and returns a success response containing a unique reference identifier for the new application.
2. **Given** a loan amount that is a whole multiple of 100 and a tenure within 3–48 months, **When** the request is submitted along with valid customer and collateral data, **Then** the submission is accepted and stored.
3. **Given** an applicant who is at least 18 years old with a positive monthly income, **When** the request is submitted, **Then** the submission is accepted and stored.
4. **Given** a customer whose marital status is "MARRIED" and whose request includes a valid spouse full name, ID number, phone number, and monthly income, **When** the request is submitted, **Then** the submission is accepted and stored together with the customer, collateral, and loan data.

---

### User Story 2 - Reject submissions with missing or malformed data (Priority: P2)

A caller submits a request that is missing a required field (e.g., no email, no license plate) or has a field in the wrong format (e.g., not-a-date birth date, non-numeric income). The system must reject the whole submission, explain what is wrong, and store nothing.

**Why this priority**: Protecting data quality is essential — bad or incomplete records must never enter the system — but it depends on Story 1's submission flow already existing to reject against.

**Independent Test**: Submit requests each missing one required field, or with one field in an invalid format, and verify each is rejected with a clear indication of the offending field(s) and that no record is created.

**Acceptance Scenarios**:

1. **Given** a request missing any required customer, collateral, or loan field, **When** it is submitted, **Then** the system rejects the request, identifies the missing field(s), and creates no record.
2. **Given** a request with an invalid email address, phone number, or birth date, **When** it is submitted, **Then** the system rejects the request, identifies the invalid field(s), and creates no record.
3. **Given** an otherwise valid request, **When** it is submitted twice with one field corrected the second time, **Then** the first submission is rejected with no record created and the second (fully valid) submission is accepted and stored.
4. **Given** a customer whose marital status is "MARRIED" but whose request is missing one or more spouse fields (full name, ID number, phone number, monthly income) or has one in an invalid format, **When** the request is submitted, **Then** the system rejects the request, identifies the invalid spouse field(s), and creates no record.

---

### User Story 3 - Enforce loan term business rules independently of field formatting (Priority: P3)

A caller submits a request where every field is well-formed, but the proposed loan amount is not a multiple of 100, or the tenure falls outside the 3–48 month range. These are business-rule violations distinct from basic field validation and must be caught even when the rest of the payload is otherwise perfect.

**Why this priority**: This protects the product's core lending rules. It is lower priority than general field validation because it covers a narrower, specific set of inputs, but it is still required before the feature can be considered complete.

**Independent Test**: Submit well-formed requests where only the loan amount is not a multiple of 100, or only the tenure is outside 3–48 months, and verify each is rejected with no record created, while confirming that boundary-valid values (tenure of exactly 3 or 48 months, an amount that is exactly divisible by 100) are accepted.

**Acceptance Scenarios**:

1. **Given** a loan amount that is not a whole multiple of 100 (e.g., 1050), **When** the request is submitted with otherwise valid data, **Then** the system rejects the request, identifies the loan amount as invalid, and creates no record.
2. **Given** a tenure outside the 3–48 month range (e.g., 0, 2, 49, or 60), **When** the request is submitted with otherwise valid data, **Then** the system rejects the request, identifies the tenure as invalid, and creates no record.
3. **Given** a tenure of exactly 3 months or exactly 48 months, **When** the request is submitted with otherwise valid data, **Then** the submission is accepted and stored.

---

### Edge Cases

- What happens when the applicant's birth date indicates an age under 18, or the birth date is in the future, or is not a real calendar date?
- A manufacturing year in the future, or earlier than 1900, is rejected as invalid (see FR-008).
- What happens when the monthly income is zero, negative, or non-numeric?
- What happens when the loan amount is zero or negative?
- How does the system handle a request where an entire section (customer, collateral, or loan) is absent from the payload?
- A request containing any field not part of the documented customer (including marital status and spouse), collateral, and loan schema is rejected outright (see FR-018).
- How does the system handle a request with correct field types but values at extreme boundaries (e.g., an extremely large loan amount or income)?
- What happens when the car's brand, model, or license plate is an unusually long non-empty string? It is accepted regardless of length, since no maximum length restriction applies (see FR-009).
- What happens when the same customer (same ID number) submits more than one loan application?
- What happens when the customer's marital status is "MARRIED" but the spouse's full name, ID number, phone number, or monthly income is missing or invalid (see FR-020)?
- What happens when the customer's marital status is not "MARRIED" but spouse data is included anyway? It is accepted and stored like any other valid field — no rejection occurs solely on that basis (see FR-023).

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST expose an operation for submitting a loan application in a single request containing customer data, collateral data, and proposed loan data.
- **FR-002**: System MUST require the following customer fields: full name, ID number, address (street, city, zip code), birth date, phone number, email address, marital status, and current monthly income.
- **FR-003**: System MUST require the following collateral fields, describing the car offered as collateral: brand, model, manufacturing year, and license plate.
- **FR-004**: System MUST require the following proposed loan fields: amount and tenure (in months).
- **FR-005**: System MUST validate that the applicant's birth date is a genuine calendar date in the past and that the applicant is at least 18 years old as of the submission date.
- **FR-006**: System MUST validate that the email address is in a well-formed email format and the phone number is in strict E.164 format (a leading `+`, country code, then digits only, no spaces or punctuation — e.g., `+14155552671`).
- **FR-007**: System MUST validate that the current monthly income is a positive numeric value, optionally with up to 2 decimal places (e.g., 4500.50); values with more than 2 decimal places or non-numeric input MUST be rejected.
- **FR-008**: System MUST validate that the car's manufacturing year is a genuine year no earlier than 1900 and not later than the current year.
- **FR-009**: System MUST validate that the license plate, car brand, car model, full name, ID number, and all address fields are non-empty; none of these free-text fields has a maximum length restriction.
- **FR-010**: System MUST validate that the proposed loan amount is a positive number, optionally expressed with up to 2 decimal places, whose numeric value is a whole multiple of 100 (e.g., 300.00 is valid; 300.50 is not, since it is not a multiple of 100).
- **FR-011**: System MUST validate that the proposed tenure is a whole number of months between 3 and 48, inclusive.
- **FR-012**: When any required field is missing or fails validation, the system MUST reject the entire submission as a single unit, MUST NOT persist any part of it, and MUST return a response identifying which field(s) failed and why.
- **FR-013**: When all customer, collateral, and loan fields pass validation, the system MUST persist the customer data, collateral data, and loan data together as one loan application record.
- **FR-014**: Every newly persisted loan application MUST be recorded with a status of "IN_PROGRESS"; the system MUST NOT allow a new submission to be created with any other initial status.
- **FR-015**: System MUST record the date and time at which a loan application was submitted.
- **FR-016**: On successful submission, the system MUST return a response containing a unique reference identifier for the newly created loan application.
- **FR-017**: System MUST NOT reject or deduplicate a submission solely because the same ID number or license plate already appears on a prior loan application.
- **FR-018**: System MUST reject the entire submission as invalid, persisting nothing, if the request payload contains any field not part of the documented customer (including marital status and spouse), collateral, or loan schema (strict schema validation, no tolerance for unrecognized fields).
- **FR-019**: System MUST validate that marital status is one of `SINGLE`, `MARRIED`, `DIVORCED`, or `WIDOWED`.
- **FR-020**: When marital status is `MARRIED`, the system MUST require the spouse's full name, ID number, phone number, and monthly income to be present and valid; if any of these spouse fields is missing or fails validation while marital status is `MARRIED`, the system MUST reject the entire submission per FR-012.
- **FR-021**: System MUST validate the spouse's full name and ID number as non-empty free text, and the spouse's monthly income using the same rule as FR-007 (a positive numeric value, optionally with up to 2 decimal places).
- **FR-022**: System MUST validate the spouse's phone number as a non-empty string containing only digits, spaces, and the characters `+`, `-`, `(`, `)` — a looser format than the strict E.164 rule FR-006 applies to the customer's own phone number.
- **FR-023**: When marital status is not `MARRIED`, spouse data is OPTIONAL; if the request includes it anyway, the system MUST validate it per FR-021 and FR-022 and persist it with the loan application record like any other accepted field, and MUST NOT reject the submission solely because spouse data is present alongside a non-`MARRIED` marital status.

### Key Entities

- **Loan Application**: A single submission that ties one customer, one collateral vehicle, and one proposed loan together. Key attributes: unique reference identifier, status (initially "IN_PROGRESS"), submission date/time.
- **Customer**: The person applying for the loan. Key attributes: full name, ID number, address (street, city, zip code), birth date, phone number, email address, marital status, current monthly income.
- **Spouse**: The customer's spouse; required when the customer's marital status is "MARRIED", optional otherwise. Key attributes: full name, ID number, phone number, monthly income.
- **Collateral (Vehicle)**: The car pledged as security for the loan. Key attributes: brand, model, manufacturing year, license plate.
- **Loan Proposal**: The requested loan terms. Key attributes: amount, tenure (months).

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A caller submitting a complete and valid loan application receives a confirmation response, including a reference identifier, in under 2 seconds.
- **SC-002**: 100% of submissions containing a missing required field, a malformed field, or a loan-term rule violation (amount not a multiple of 100, or tenure outside 3–48 months) are rejected with no record created.
- **SC-003**: 100% of accepted submissions result in exactly one stored loan application record with status "IN_PROGRESS" and all submitted data preserved without alteration.
- **SC-004**: For every rejected submission, the caller can identify which specific field(s) caused the rejection from the response alone, without needing outside assistance.
- **SC-005**: The system correctly accepts or rejects loan term boundary values (tenure of 3 and 48 months; amounts exactly divisible by 100) with 100% accuracy.

## Assumptions

- The applicant must be at least 18 years old at the time of submission; no explicit minimum age was given, so the standard legal age of majority is assumed.
- There is no maximum cap on the proposed loan amount beyond it being a positive multiple of 100; income-to-loan ratio checks and affordability scoring are out of scope for this feature.
- Callers of this API (e.g., an internal loan-origination front-end or partner system) are already authenticated through the organization's existing authentication mechanism; the specific authentication scheme is out of scope for this feature.
- The system does not enforce uniqueness across submissions — the same customer (by ID number) or the same vehicle (by license plate) may appear on more than one loan application.
- Reviewing, approving, rejecting, or otherwise transitioning a loan application out of "IN_PROGRESS" status is out of scope for this feature; this feature covers submission and initial storage only.
- Monthly income and loan amount are expressed in a single, implicit currency; multi-currency support is out of scope.
- The full name, ID number, address fields, brand, model, and license plate are treated as free-text strings; no country-specific format validation (e.g., a specific national ID or plate format), reference-list validation (e.g., a fixed list of valid car manufacturers/models), or maximum length restriction is assumed beyond non-emptiness. The phone number is the exception: the customer's own phone number MUST conform to strict E.164 format (see FR-006), while the spouse's phone number uses a looser, non-E.164 format (see FR-022).
- The spouse's monthly income is stored for record-keeping only; it is not combined with the customer's income for affordability or eligibility calculations, consistent with affordability scoring being out of scope for this feature.
