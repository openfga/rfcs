# Meta

* Name: Idempotent Writes and Deletes
* Start Date: 2025-08-15
* Author(s): @cikasfm, @adriantam
* Status: Draft
* RFC Pull Request: (leave blank)
* Relevant Issues:
  - [#79 Idempotent Writes](https://github.com/openfga/roadmap/issues/79)
* Supersedes: N/A

-----

# Summary

This RFC proposes adding idempotent write and delete functionality to the OpenFGA API. This feature will allow API requests to succeed even if some of the tuples being written already exist or some of the tuples being deleted do not exist. The solution introduces optional `on_duplicate` and `on_missing` flags in the `Write` API request, providing a simple and backward-compatible way for developers to avoid complex client-side logic and unnecessary `400 Bad Request` errors.

-----

# Definitions

* **Idempotent Write:** An operation that can be executed multiple times without changing the result beyond the initial execution. In this context, a write request that contains a tuple that already exists will not fail but will be treated as a successful "no-op" for that specific tuple.
* **No-op (No-operation):** An instruction that does nothing. When a client requests to write a tuple that already exists with the `on_duplicate: ignore` flag, the system will perform a no-op on that tuple and proceed with the rest of the request.
* **Tuple Conflict:** Occurs when a write request attempts to insert a tuple that already exists but with a different set of attributes (e.g., a different `condition`). The proposed solution will not resolve these conflicts and will continue to return an error.

-----

# Motivation

* **Why should we do this?** The current OpenFGA API behavior, where a single duplicate tuple fails an entire batch `Write` request, is a significant pain point for developers. This behavior is a common source of frustration, leading to complex and brittle client-side code. By introducing idempotent behavior, we can dramatically improve the developer experience and reduce the volume of errors developers must handle.
* **What use cases does it support?** This feature simplifies several common use cases, including:
  * **Data Synchronization:** Makes it easier to synchronize data from external sources without having to first query OpenFGA to check for existing tuples.
  * **Retry Mechanisms:** Allows for simple and safe retry logic on failed network requests without needing to de-duplicate the request payload.
  * **Batch Operations:** Enables developers to send large batches of writes more reliably and efficiently, as the entire request will not fail due to a single duplicate entry.
* **What is the expected outcome?** We expect a significant reduction in `400 Bad Request` errors caused by duplicate writes. This will lead to a more robust and scalable API for clients, simplifying their application logic and reducing overall friction.

-----

# What it is

This feature introduces two optional parameters to the `Write` API request, which allows an application developer to specify the desired behavior for duplicate tuples and non-existent tuples.

* **Target Persona:** This feature is primarily for the **application developer** and **platform operator** personas, as it directly impacts client-side logic for interacting with the API.

* **New Terminology:**

  * `on_duplicate`: A flag to control behavior for writes of existing tuples.
  * `on_missing`: A flag to control behavior for deletes of non-existent tuples.

* **Behavioral Overview:**
  The core of the feature lies in the behavior of the new flags. When a request is sent with `on_duplicate: ignore` and a tuple within the request already exists, the API will not return an error. Instead, it will effectively "ignore" the write for that specific tuple and continue processing the rest of the request. Similarly, with `on_missing: ignore`, the API will not return an error if a tuple to be deleted is not found in the database.

* **Example Request:**
  The following example demonstrates how the new parameters would be used in a `Write` request:

```json
{
  "writes": {
    "tuple_keys": [
      {
        "user": "user:anne",
        "relation": "writer",
        "object": "document:2021-budget"
      }
    ],
    "on_duplicate": "ignore"
  },
  "deletes": {
    "tuple_keys": [
      {
        "user": "user:bob",
        "relation": "reader",
        "object": "document:2021-budget"
      }
    ],
    "on_missing": "ignore"
  },
  "authorization_model_id": "01G50QVV17PECNVAHX1GG4Y5NC"
}
```

  If the tuple for `user:anne` already exists and the tuple for `user:bob` does not exist, this request will succeed with no error returned.

-----

# How it Works

The implementation of this feature will be handled within a single, atomic database transaction to ensure data integrity and prevent race conditions. The process for a single `Write` API call is as follows:

1.  **Start Transaction:** A database transaction is initiated with a `READ COMMITTED` isolation level.
2.  **Lock Tuples:** This step is only performed if at least one of the `on_duplicate: ignore` or `on_missing: ignore` flags is set. In that case, a single `SELECT ... FOR UPDATE` query is compiled and executed. This query will read and lock all tuples specified in both the `writes` and `deletes` arrays. This is the critical step for mitigating race conditions, as it prevents other transactions from modifying these rows during our operation.
3.  **Process Deletes:** The application logic iterates through the `deletes` list. For each tuple:
* If `on_missing: error` is set, the system will execute the delete statement and check the number of rows affected. If no rows are affected, the transaction is rolled back and an error is returned.
* If `on_missing: ignore` is set, the system simply creates a batch `DELETE` statement for all tuples that were found in the locked set. Tuples that were not found are ignored as a no-op.
4.  **Process Writes:** The application logic iterates through the `writes` list. For each tuple:
* If `on_duplicate: error` is set, the system will execute the insert statement. If a `CONSTRAINT VIOLATION` error occurs, the transaction is rolled back and an error is returned.
* If `on_duplicate: ignore` is set, the system verifies that the tuple is not a "conflict" (i.e., it has the same key but different `condition` data). If it is a conflict, the transaction is rolled back. Otherwise, if the tuple does not exist in the locked set, it is added to a batch `INSERT` statement.
5.  **Execute Batches:** The single, batched `DELETE` statement and the single, batched `INSERT` statement are executed within the transaction.
6.  **Update Changelog:** The system inserts changelog entries only for the tuples that were successfully inserted or deleted.
7.  **Commit Transaction:** The transaction is committed.

This approach is a significant optimization over the existing implementation, which performs individual database round-trips for each tuple. The new approach reduces the total number of round-trips to a handful, improving overall API efficiency.

-----

# Migration

This proposal is designed to be fully backward-compatible.

* **Application Developers:** No changes are required for existing applications. The current default behavior (`on_duplicate: error`, `on_missing: error`) will be preserved if the new flags are not specified in the API request. Developers can adopt the new flags at their convenience to simplify their code.
* **Authorization Platform Operators:** No migration of existing data or systems is needed. The update will be a drop-in change to the OpenFGA service.
* **Compatibility:** There are no planned breaks to the public API.

-----

# Drawbacks

* **Lack of Granularity:** The primary drawback of the chosen approach is the lack of per-tuple control. A developer cannot, for example, have one tuple in a request that uses `on_duplicate: ignore` while another tuple in the same request uses the default `on_duplicate: error`. This is a limitation that may be addressed in future work.
* **Database Complexity:** The proposed `SELECT ... FOR UPDATE` pattern adds complexity to the datastore logic, requiring careful implementation to avoid potential deadlocks, though this is a standard and well-understood pattern.

-----

# Alternatives

* **Option 2: Per-Tuple Operation Parameter:** This approach would have offered more fine-grained control, allowing developers to specify `CREATE`, `TOUCH`, or `DELETE` for each individual tuple in the request. This would be a more flexible and robust solution long-term.
* **Why is this proposal the best?** The chosen proposal (Option 1) was selected because it is a simpler, less intrusive change to the API that solves the most pressing developer pain point with minimal implementation effort. It provides the best balance of simplicity and a direct fix to the problem, allowing us to deliver value faster.
* **What is the impact of not doing this?** Without this feature, developers will continue to face the challenges of `400 Bad Request` errors on every duplicate write, forcing them to implement complex and inefficient client-side logic to ensure data consistency. This will hinder OpenFGA's adoption and competitive position.

-----

# Prior Art

Competitors in the authorization space, such as SpiceDB already offer idempotent write functionality. This feature aligns OpenFGA with industry standards and best practices, addressing a common developer need.

-----

# Questions

**Q: What is the final performance validation plan to ensure the new datastore logic does not introduce significant overhead?**
* If the new `on_duplicate: ignore` or `on_missing: ignore` parameters are not used, no additional `SELECT` operation will be performed. Additionally, the new implementation will consolidate multiple individual delete and insert requests (up to 40) into just two batched requests (one for deletes, one for inserts). This change alone should significantly **improve performance** over the current, unoptimized flow.
* When an `ignore` parameter is used, a single `SELECT FOR UPDATE` will be added to the call, which includes an extra database read and a lock. While this adds one round trip, the consolidation of individual delete and insert requests means that the overall number of database round trips should remain a net positive, so performance is not expected to be worse than before.

---
**Q: How will we collaborate with the community member who has already submitted a PR on this topic?**

**A:** This public RFC serves as the answer to the issue [#79 Idempotent Writes](https://github.com/openfga/roadmap/issues/79). It formalizes the design and will be linked to the existing PR [#2593](https://github.com/openfga/openfga/pull/2593) for community review and collaboration, ensuring the contributor's work is recognized and integrated into the official design process.

---

**Q: Will idempotent operations affect the generation of consistency tokens (Zookies)?**

**A:** We don't expect any change, Zookies are out of scope.