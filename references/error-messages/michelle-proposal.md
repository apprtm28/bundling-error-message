Here is the converted Markdown format of the "Scenario" sheet:

``` markdown
Validation to vertical will be done sequentially
Error message will be displayed to users sequentially

Sequence prioritization within each vertical:
1. Non-business related error
2. Inventory related error
3. Non-price related error
*Price related error will not be shown until both Flight and Accommodation final check is successful
*Definition of successful: No non-business error and inventory is still available

Sequence prioritization across vertical (when both Flight and Accommodation final check is successful):
1. Price related error
2. Accommodation non-price related error
*Flight non-price related error will have already been displayed before Accommodation final check is successful

| Accommodation | | Flight (Non-business related error) | Flight (Inventory related error) | Flight (Non-price related error) | Flight (Price related error) |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Accommodation** | **Non-business error** | 1. Display Flight non-business related error as-is<br>2. Do final check to Flight service again<br>*Don't do validation for Accommodation items until Flight final check is successful | 1. Display Flight inventory related error as-is<br>2. Redirect user to Bundling SRP > Bundling service will re-do the pre-defined Flight selection | 1. Display Flight non-price related error as-is<br>2. Display Accommodation non-business related error as-is<br>3. Do final check to Accommodation service again | 1. Display Accommodation non-business related error as-is<br>2. Do final check to Accommodation service again |
| **Accommodation** | **Inventory related error** | 1. Display Flight non-business related error as-is<br>2. Do final check to Flight service again<br>*Don't do validation for Accommodation items until Flight final check is successful | 1. Display Flight inventory related error as-is<br>2. Redirect user to Bundling SRP > Bundling service will re-do the pre-defined Flight selection | 1. Display Flight non-price related error as-is<br>2. Display Accommodation inventory related error as-is<br>3. Redirect user to Bundling SRP | 1. Display Accommodation inventory related error as-is<br>2. Redirect user to Bundling SRP |
| **Accommodation** | **Non-price related error (single or multiple)** | 1. Display Flight non-business related error as-is<br>2. Do final check to Flight service again<br>*Don't do validation for Accommodation items until Flight final check is successful | 1. Display Flight inventory related error as-is<br>2. Redirect user to Bundling SRP > Bundling service will re-do the pre-defined Flight selection | 1. Display Flight non-price related error as-is<br>2. Display Accommodation non-price related error as-is<br>3. Redirect user to Payment Page | 1. Display Flight price related error in OPAQUE PRICE<br>2. Display Accommodation non-price related error as-is<br>3. Redirect user to Payment Page |
| **Accommodation** | **Price related error** | 1. Display Flight non-business related error as-is<br>2. Do final check to Flight service again<br>*Don't do validation for Accommodation items until Flight final check is successful | 1. Display Flight inventory related error as-is<br>2. Redirect user to Bundling SRP > Bundling service will re-do the pre-defined Flight selection | 1. Display Flight non-price related error as-is<br>2. Display Accommodation price related error in OPAQUE PRICE<br>3. Redirect user to Payment Page | 1. Display Flight and Accommodation price related error in OPAQUE PRICE<br>2. Redirect user to Payment Page |
| **Accommodation** | **Combination of non-price related error and price related error** | 1. Display Flight non-business related error as-is<br>2. Do final check to Flight service again<br>*Don't do validation for Accommodation items until Flight final check is successful | 1. Display Flight inventory related error as-is<br>2. Redirect user to Bundling SRP > Bundling service will re-do the pre-defined Flight selection | 1. Display Flight non-price related error as-is<br>2. Display Accommodation price related error in OPAQUE PRICE<br>3. Display Accommodation non-price related error as-is *although displayed as-is, price related error should be stripped from the combination<br>4. Redirect user to Payment Page | 1. Display Flight and Accommodation price related error in OPAQUE PRICE<br>2. Display Accommodation non-price related error as-is *although displayed as-is, price related error should be stripped from the combination<br>3. Redirect user to Payment Page |

```
