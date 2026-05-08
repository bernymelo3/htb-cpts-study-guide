## ID
205

## Module
Using Web Proxies

## Kind
notes

## Title
Section 8 — Encoding/Decoding

## Description
Hands‑on walkthrough of using built‑in encoder/decoder tools in Burp Suite and ZAP for URL encoding, Base64, and other formats critical for tampering with HTTP requests.

## Tags
encoding, decoding, burp-suite, zap, base64, url-encoding

## Commands
- In Burp Repeater, select text → right-click → Convert Selection → URL → URL-encode key characters
- In Burp, press Ctrl+U to URL-encode selected text while typing
- In Burp, go to the **Decoder** tab to encode/decode using multiple methods (Base64, HTML, Unicode, etc.)
- In ZAP, press Ctrl+E to open the **Encoder/Decoder/Hash** tool
- In ZAP, paste encoded text into the Encode/Decode/Hash dialog; it auto‑decodes across multiple decoders in parallel
- In ZAP, use the **Add New Tab** button to create a custom tab with your preferred set of encoders/decoders
- In Burp Inspector, right-click a value and select **Decode as…** to apply decoding on the fly

## What This Section Covers
HTTP requests and responses frequently contain encoded data — URL‑encoded parameters, Base64‑encoded cookies, or HTML entities. Manually encoding/decoding these values is essential when modifying requests to test for vulnerabilities. Both Burp Suite and ZAP provide powerful built‑in tools that let you quickly switch between encoding schemes without leaving the proxy.

## Methodology
1. In Burp Repeater (or any editor), highlight a string and right-click → **Convert Selection → URL → URL-encode key characters** to encode spaces, `&`, `#`, and other dangerous characters.
2. Alternatively, enable **URL-encode as you type** by right-clicking and toggling the option — any subsequent typing will be automatically encoded.
3. For other encodings (Base64, HTML, Unicode, ASCII hex), open the **Decoder** tab.
4. Paste the encoded string, then choose **Decode as…** from the dropdown or right-click menu. The decoded result appears in a lower pane.
5. To re‑encode modified data, paste the plaintext in the input area, select **Encode as…** (e.g., Base64), and copy the output.
6. In ZAP, press **Ctrl+E** to open **Encoder/Decoder/Hash**. Paste a string; it will appear in both the *Encode* and *Decode* tabs, with multiple decoders applied automatically.
7. In the *Decode* tab, the tool tries various decodings (Base64, URL, Hex, etc.) and shows the result for each. You can immediately see which one produces readable text.
8. To encode, go to the *Encode* tab, type or paste the text, and select the encoder from the dropdown. Copy the encoded string back to the Request Editor.
9. Customise ZAP’s tool: click **Add New Tab** to create a view that shows only the encoders/decoders you need (e.g., Base64 + URL).
10. In newer Burp versions, use the **Inspector** (available in Proxy and Repeater) to right-click any parameter and directly apply encoding or decoding without switching tabs.

## Multi-step Workflow (optional)

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|web-proxies]]  
← [[07-repeating-requests]] | [[09-proxying-tools]] →
<!-- AUTO-LINKS-END -->
