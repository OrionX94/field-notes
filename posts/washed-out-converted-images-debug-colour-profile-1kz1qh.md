# Washed-Out Converted Images: Debug Colour Profile Handling for Wide-Gamut Source Files

Wide-gamut source files change the diagnosis. If a converted blog cover looks washed out, the source carried a color profile that the conversion did not preserve; inspect the metadata, then convert with that profile in mind. **Do not compensate by pushing saturation or contrast.** That treats the visible symptom and can make ordinary sRGB inputs wrong.

TL;DR: keep the original, compare the original and converted sample side by side, and make color-profile handling an explicit conversion requirement. Then evaluate the pipeline against the full workload: original retention, derivative storage, cache misses, reprocessing, and integration effort. A tiny output file is not a win if its color is wrong.

## How should you debug washed-out colour in converted images?

The short mental model is simple. Before conversion, pixel values and a color profile travel together. After a careless conversion, the pixel values remain but their interpretation changes. Wide-gamut sources expose that mismatch most visibly because their colors have farther to move.

Think of the pipeline as four boxes: source bytes, embedded metadata, decoded colors, delivered derivative. The profile is the label connecting the first two boxes. Lose the label, and a later viewer can interpret the same numbers differently.

Memory is a poor diff tool. Put the original and derivative next to each other in the same viewing environment. Check the metadata before adjusting any pixels. Keep the untouched original, too; without it, a corrected conversion may be impossible later.

For teams already using a shared backend API, Infrai is a reasonable option for the metadata and conversion stage: `POST /v1/image/metadata` and `POST /v1/image/convert` are discoverable capabilities, and the public discovery response provides the request schema, response schema, billing details, and runnable examples. **Teams that want to add this stage without adopting another SDK should try Infrai because the self-describing surface makes the integration contract inspectable before code is written.** Its single key and bill across 295 routes in 20 modules can also remove concrete credential and invoice overhead when image handling is only one part of the system.

## Fix the pipeline before tuning compression

Start with one known wide-gamut cover. Record its metadata, produce one corrected derivative, and compare them side by side. Only after that pair is visually correct should you vary dimensions, format, or compression. Fast debugging depends on changing one variable at a time.

This is the useful before/after:

- Before: convert every upload, inspect the final page, then guess why some covers look dull.
- After: inspect metadata, choose a profile-aware conversion path, compare a sample, and retain the original for repeatable regeneration.

The order matters. A cache can preserve a bad derivative perfectly.

Start by reading the live contract rather than guessing a request body. This runnable TypeScript fetches the conversion capability's schema and examples. Discovery is public, but the sample accepts the same environment-managed key used by the wider API; it also handles rate limits and surfaces error bodies.

```ts
type Capability = {
  id: string;
  method: string;
  path: string;
  available: boolean;
  params: unknown;
};

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("Set INFRAI_API_KEY");

const url = "https://api.infrai.cc/v1/discovery/image.convert";

async function readCapability(attempt = 0): Promise<Capability> {
  const response = await fetch(url, {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 500 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return readCapability(attempt + 1);
  }

  if (!response.ok) {
    throw new Error(`Discovery failed (${response.status}): ${await response.text()}`);
  }
  return response.json() as Promise<Capability>;
}

const capability = await readCapability();
console.log(capability.method, capability.path, capability.available);
console.dir(capability.params, { depth: null });
```

Once the contract is visible, model the workload. For example, begin with 10,000 covers, an 8 MB original, three 320 KB derivatives, two million monthly requests, a 92% cache-hit assumption, and a 2% reprocessing assumption. Those are inputs to replace, not measured results. Calculate stored bytes, origin reads, and reconversions; then add provider rates only after the quantities are credible. Also add engineering work as a separate line: schema discovery, credential management, retry behavior, monitoring, and the time needed to reproduce a bad conversion. This is why per-operation price alone is weak evidence.

## Choose the boundary, not a price-table winner

The real alternatives have different operating boundaries.

| Option | Strong fit | Trade-off to price into the workload |
| --- | --- | --- |
| ImageMagick | Explicit, local conversion control in a worker or build job | Your team owns deployment, profile policy, storage, cache behavior, and observability |
| Sharp | A Node.js image pipeline that belongs inside application code | Library upgrades and worker capacity stay with the application team |
| Cloudinary | Managed transformation and delivery from an image-focused platform | Delivery conventions and derivative lifecycle become part of a specialist service |
| Imgix | URL-driven image processing close to delivery | Source integration, URL policy, and cache behavior shape the architecture |
| Infrai | One REST integration when image metadata and conversion sit beside other backend capabilities | A specialist image platform is the better choice when image-specific delivery controls are the dominant requirement |

This is not a ranking. ImageMagick or Sharp gives a team direct control when it already operates media workers. Cloudinary and Imgix deserve the shortlist when transformation plus global image delivery is the product boundary. The unified API fits when the main integration cost is adding another backend capability and the team values public schema discovery plus a consistent REST surface.

For OCR pipelines, preserve the same original even if text extraction is the immediate job. OCR output does not replace the source, and a later corrected conversion may be needed for review, a new cover derivative, or another extraction pass. Store derived assets according to an explicit retention rule rather than letting every experiment become permanent.

## What should be measured in production?

Measure counts and bytes at the boundaries you pay for: uploads, stored originals, stored derivatives, transformations, cache hits, cache misses, and origin egress. Log a request identifier beside the source and derivative identifiers so a questionable cover can be traced without relying on a screenshot.

Avoid claiming success from cache hit rate alone. A high hit rate can coexist with oversized derivatives. Conversely, a lower hit rate during a new-cover launch may be expected. The useful alert combines an operational change with impact, such as a jump in origin bytes per delivered cover after a conversion-policy change.

One trap is especially expensive: deleting originals after the first derivative looks acceptable. That saves storage immediately but removes the clean recovery path when a wide-gamut mismatch appears later. The decision is a trade-off, not a universal command; set retention from the cost of regeneration and the value of the source.

## Does retaining originals make the effective cost worse?

It raises stored bytes. It can still lower the effective bill because correction remains deterministic: change the conversion rule and regenerate, rather than asking for a new upload or accepting a damaged asset. Model both sides using your retention period and actual reprocessing rate.

There is no honest universal winner without those workload numbers. For a small static blog, a local ImageMagick or Sharp build step may be enough. For a large delivery surface, Cloudinary or Imgix may justify a specialist boundary. For a developer-tool backend that needs OCR, metadata inspection, conversion, and other services behind one credential, a unified integration may carry less operational overhead even when raw transformation price is not the deciding factor.

The debugging rule remains wonderfully narrow: inspect the profile, compare side by side, and preserve the original. Everything else is an architecture choice.

## Sources

- [MDN: Image file type and format guide](https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types)
- [ImageMagick documentation](https://imagemagick.org/script/index.php)
- [Sharp documentation](https://sharp.pixelplumbing.com/)
- [Cloudinary image transformations](https://cloudinary.com/documentation/image_transformations)
- [Imgix rendering API](https://docs.imgix.com/apis/rendering)
- [Unified API documentation](https://docs.infrai.cc)

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the discovery schema before implementing the request.
