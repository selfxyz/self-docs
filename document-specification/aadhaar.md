# Aadhaar

## What is Aadhaar?&#x20;

Aadhaar is India’s national digital identity system maintained by UIDAI (Unique Identification Authority of India).&#x20;

* It is a **12-digit unique identity number** issued to residents of India, based on their biometric and demographic data.
* Aadhaar serves as a **foundational ID** that can be used to verify identity across financial, telecom, and government services.
* Verification is done via UIDAI APIs or Aadhaar-linked services, such as QR codes.

{% hint style="danger" %}
UIDAI does not sign the entire Aadhaar number and only the last 4 digits. As this does not have enough entropy to create a nullifer we also use other fields such as the name, date of birth, and gender. This also means that person can change their Aadhaar and be considered as a "new" person in the protocol. Although we have chosen fields that are hard to change edge cases do pop up such as people changing their names due to marriage, divorce or other legal corrections.&#x20;
{% endhint %}

## Registering with Aadhaar on Self

Self allows users to prove identity using Aadhaar in a **privacy-preserving** way. There are two common methods:

1. #### Using the mAadhaar app&#x20;
   1. #### Download the official **mAadhaar app** from UIDAI ([Android](https://play.google.com/store/apps/details?id=in.gov.uidai.mAadhaarPlus)/[IOS](https://apps.apple.com/in/app/maadhaar/id1435469474)).&#x20;
   2.  #### Inside the app, generate a **QR code** that contains Aadhaar demographic details.

       <figure><img src="../.gitbook/assets/WhatsApp Image 2025-09-26 at 00.13.52.jpeg" alt="" width="149"><figcaption><p>mAadhaar</p></figcaption></figure>
   3. #### Upload this QR code in the Self app to register.
      1. #### Self processes this QR to derive a unique **nullifier** (an identifier) that allows users to prove uniqueness without sharing their Aadhaar number directly.
2. #### By using the UIDAI website
   1. #### Go to the [UIDAI website](https://myaadhaar.uidai.gov.in/genricDownloadAadhaar/en)
   2. #### Enter your Aadhaar number, and verify yourself with an OTP to get a PDF.&#x20;
   3. #### Take a screenshot of the QR code in the pdf and upload in the Self app.&#x20;

{% hint style="info" %}
The password to the PDF in both cases is the first four letters of your name in caps following by your birth year.
{% endhint %}

## Working with attributes in Aadhaar

Since Aadhaar uses a different format than passports, keep in mind that some of the disclosed attribute formats can be different.&#x20;

### `GenericDiscloseOutputV2`&#x20;

<table><thead><tr><th>Field </th><th>Type</th><th>Formatting / Value</th></tr></thead><tbody><tr><td><pre><code>forbiddenCountriesListPacked
</code></pre></td><td><code>uint256[4]</code></td><td>Packed bytes of all countries into an array 4 <code>uint256</code> numbers.</td></tr><tr><td><pre><code>issuingState
</code></pre></td><td><code>string</code></td><td>Varies**</td></tr><tr><td><pre><code>name
</code></pre></td><td><code>string[]</code></td><td>First value contains the entire name (and it may vary)</td></tr><tr><td><pre><code>idNumber
</code></pre></td><td><code>string</code></td><td>Last 4 letters of the aadhaar number. </td></tr><tr><td><pre><code>nationality
</code></pre></td><td><code>string</code></td><td><code>'IND'</code></td></tr><tr><td><pre><code>dateOfBirth
</code></pre></td><td><code>string</code></td><td><code>DD-MM-YYYY</code></td></tr><tr><td><pre><code>gender
</code></pre></td><td><code>string</code></td><td><code>'M' | 'T' | 'F'</code></td></tr><tr><td><pre><code>expiryDate
</code></pre></td><td><code>string</code></td><td><code>'UNAVAILBLE'</code></td></tr><tr><td><pre><code>olderThan
</code></pre></td><td><code>uint256</code></td><td><code>0 - 99</code></td></tr><tr><td><pre><code>ofac
</code></pre></td><td><code>bool[3]</code></td><td>Each value is true if ofac check is enabled.</td></tr></tbody></table>

### `GenericDiscloseOutput`

<table><thead><tr><th>Field</th><th>Type</th><th>Formatting / Value</th></tr></thead><tbody><tr><td><pre><code>forbiddenCountriesListPacked
</code></pre></td><td><code>string[]</code></td><td>Packed bytes list of forbidden countries</td></tr><tr><td><pre><code>issuingState
</code></pre></td><td><code>string</code></td><td>Varies</td></tr><tr><td><pre><code>name
</code></pre></td><td><code>string</code></td><td>Full name. Can vary as well</td></tr><tr><td><pre><code>idNumber
</code></pre></td><td><code>string</code></td><td>Last 4 digits of aadhaar</td></tr><tr><td><pre><code>nationality
</code></pre></td><td><code>string</code></td><td><code>'IND'</code></td></tr><tr><td><pre><code>dateOfBirth
</code></pre></td><td><code>string</code></td><td><code>YYYYMMDD</code></td></tr><tr><td><pre><code>gender
</code></pre></td><td><code>string</code></td><td><code>'M' | 'T' | 'F'</code></td></tr><tr><td><pre><code>expiryDate
</code></pre></td><td><code>string</code></td><td><code>'UNAVAILABLE'</code></td></tr><tr><td><pre><code>minimumAge
</code></pre></td><td><code>string</code></td><td><code>'00'-'99'</code>  </td></tr><tr><td><pre><code>ofac
</code></pre></td><td><code>bool[]</code></td><td>True if in ofac</td></tr></tbody></table>

{% hint style="info" %}
\*\*Varies refers to a string being all caps / lowercase / first letter in caps and the rest in lowercase.
{% endhint %}

