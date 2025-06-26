# Utilize Passport Attributes

In the happy-birthday example, we demonstrated one way to utilize the passport information when integrating Self. Note that the passport details—and the forbiddenCountries data—are bundled into the circuit’s public signals in order to reduce onchain data. This bundled format is not very user-friendly for reading.



If you simply need the data in a readable format, you can call the following functions from the IdentityVerificationHub.



### Readable Data Extraction

#### Get Readable Passport Data

You can retrieve readable attributes by calling:

```solidity
function getReadableRevealedData(
    uint256[3] memory revealedDataPacked,
    RevealedDataType[] memory types
)
    external
    virtual
    onlyProxy
    view
    returns (ReadableRevealedData memory)
{
    bytes memory charcodes = Formatter.fieldElementsToBytes(
        revealedDataPacked
    );

    ReadableRevealedData memory attrs;

    for (uint256 i = 0; i < types.length; i++) {
        RevealedDataType dataType = types[i];
        if (dataType == RevealedDataType.ISSUING_STATE) {
            attrs.issuingState = CircuitAttributeHandler.getIssuingState(charcodes);
        } else if (dataType == RevealedDataType.NAME) {
            attrs.name = CircuitAttributeHandler.getName(charcodes);
        } else if (dataType == RevealedDataType.PASSPORT_NUMBER) {
            attrs.passportNumber = CircuitAttributeHandler.getPassportNumber(charcodes);
        } else if (dataType == RevealedDataType.NATIONALITY) {
            attrs.nationality = CircuitAttributeHandler.getNationality(charcodes);
        } else if (dataType == RevealedDataType.DATE_OF_BIRTH) {
            attrs.dateOfBirth = CircuitAttributeHandler.getDateOfBirth(charcodes);
        } else if (dataType == RevealedDataType.GENDER) {
            attrs.gender = CircuitAttributeHandler.getGender(charcodes);
        } else if (dataType == RevealedDataType.EXPIRY_DATE) {
            attrs.expiryDate = CircuitAttributeHandler.getExpiryDate(charcodes);
        } else if (dataType == RevealedDataType.OLDER_THAN) {
            attrs.olderThan = CircuitAttributeHandler.getOlderThan(charcodes);
        } else if (dataType == RevealedDataType.PASSPORT_NO_OFAC) {
            attrs.passportNoOfac = CircuitAttributeHandler.getPassportNoOfac(charcodes);
        } else if (dataType == RevealedDataType.NAME_AND_DOB_OFAC) {
            attrs.nameAndDobOfac = CircuitAttributeHandler.getNameAndDobOfac(charcodes);
        } else if (dataType == RevealedDataType.NAME_AND_YOB_OFAC) {
            attrs.nameAndYobOfac = CircuitAttributeHandler.getNameAndYobOfac(charcodes);
        }
    }

    return attrs;
}
```

#### Get Readable Forbidden Countries

Similarly, to extract the forbiddenCountries list in a readable format:

```solidity

function getReadableForbiddenCountries(
    uint256[4] memory forbiddenCountriesListPacked
)
    external
    virtual
    onlyProxy
    view
    returns (string[MAX_FORBIDDEN_COUNTRIES_LIST_LENGTH] memory)
{
    return Formatter.extractForbiddenCountriesFromPacked(forbiddenCountriesListPacked);
}
```



In the `getReadableRevealedData` function, you specify the types of data you want to expose to reduce unnecessary processing and save gas. Please use the following [enum](https://github.com/selfxyz/self/blob/9d8c475de086b316c160c98597fd37c97c8efd66/contracts/contracts/interfaces/IIdentityVerificationHubV1.sol#L19-L31) defined in the [IIdentityVerificationHub](https://github.com/selfxyz/self/blob/main/contracts/contracts/interfaces/IIdentityVerificationHubV1.sol):

```solidity
enum RevealedDataType {
    ISSUING_STATE,     // The issuing state of the passport.
    NAME,              // The full name of the passport holder.
    PASSPORT_NUMBER,   // The passport number.
    NATIONALITY,       // The nationality.
    DATE_OF_BIRTH,     // The date of birth.
    GENDER,            // The gender.
    EXPIRY_DATE,       // The passport expiry date.
    OLDER_THAN,        // The "older than" age verification value.
    PASSPORT_NO_OFAC,  // The passport number OFAC status.
    NAME_AND_DOB_OFAC, // The name and date of birth OFAC status.
    NAME_AND_YOB_OFAC  // The name and year of birth OFAC status.
}
```



### Additional Libraries for Enhanced Flexibility

Self provides several convenient functions as libraries so that external contracts can easily use our features. The two main libraries included in `@selfxyz/contracts` are **Formatter** and **CircuitAttributeHandler**. Below is a summary of their key functions:

#### Formatter

* **formatName**\
  Converts raw name data from the passport into a human-readable format.
* **formatDate**\
  Transforms date data from the YYMMDD format into the DD-MM-YY format.
* **numAsciiToUint**\
  Converts numerical data represented as ASCII characters into a uint.
* **fieldElementToBytes**\
  Converts the three bn254 field elements of `revealedDataPacked` into a single bytes array.
* **extractForbiddenCountriesFromPacked**\
  Transforms the four bn254 field elements of `forbiddenCountriesListPacked` into a readable array of three-letter country codes.
* **proofDateToUnixTimestamp**\
  Converts the six proof-generated date signals into the Unix timestamp for midnight of that day.
* **dateToUnixTimestamp**\
  Converts a date string in YYMMDD format into the Unix timestamp for midnight of that day.
* **substring**\
  Used to extract a substring from a given string.
* **parseDatePart**\
  Converts a date component represented as a string into a numeric value.
* **toTimestamp**\
  Accepts numeric year, month, and day values and returns the Unix timestamp for midnight of that day.
* **isLeapYear**\
  Checks whether a given year is a leap year when converting to a timestamp.

#### CircuitAttributeHandler

* **getIssuingState**\
  Extracts the issuing state information from the bytes array produced by `fieldElementToBytes`.
* **getName**\
  Extracts the name from the bytes data.
* **getPassportNumber**\
  Extracts the passport number from the bytes data.
* **getNationality**\
  Extracts the nationality from the bytes data.
* **getDateOfBirth**\
  Extracts the date of birth from the bytes data.
* **getGender**\
  Extracts the gender from the bytes data.
* **getExpiryDate**\
  Extracts the expiry date from the bytes data.
* **getOlderThan**\
  Extracts the age verification ("older than") value from the bytes data.
* **getPassportNoOfac**\
  Extracts the OFAC check result for the passport number from the bytes data.
* **getNameAndDobOfac**\
  Extracts the OFAC check result for the name and date of birth from the bytes data.
* **getNameAndYobOfac**\
  Extracts the OFAC check result for the name and year of birth from the bytes data.
* **compareOfac**\
  Verifies that the OFAC check passes without issues.
* **compareOlderThan**\
  Checks that the proof meets the required minimum age.
* **extractStringAttributes**\
  Extracts string attributes from specific positions within the bytes data.



Use these functions in your contracts as needed to enhance the usability of the passport data and to interact flexibly with Self’s verification functionality.

