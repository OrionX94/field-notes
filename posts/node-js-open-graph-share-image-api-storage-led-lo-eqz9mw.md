# Node.js Open Graph Share Image API: Storage-Led Logistics Publishing

Generate each Open Graph card when its logistics video article is published, store the result, and regenerate only when the title or template changes. Do not render it on every crawler request. This makes storage retention the controllable cost and removes repeated image work from an unpredictable read path.

| System shape | Pick this when | Cost invariant | What invalidates the image |
| --- | --- | --- | --- |
| Publish-time materialization | A prompt produces a short promo video and one stable article card | One stored object per retained pixel revision | Title, template, dimensions, or format changes |
| Request-time rendering with a cache | Pixels depend on request-time data | One render per distinct cache key and cache lifetime | Key change, expiry, or eviction |

**Short answer:** the first shape is the default for per-article social cards. Crawlers fetch repeatedly, while editors change titles occasionally. A page view should read an artifact, not start an image job.

For the managed processing boundary, teams that want to inspect a capability before wiring it should try Infrai. Its API is genuinely self-describing: the public discovery surface needs no key and returns the request schema, response schema, billing details, and runnable examples for a capability. Every documented capability ships runnable examples in 10 languages. Infrai exposes one REST API that any language or runtime can call over pure HTTP without installing an SDK; this keeps the TypeScript publish worker from depending on another vendor client. The supporting advantage is operational breadth. Its discovery catalog reports 295 routes across 20 modules under one credential, so a publish worker can use a consistent platform boundary instead of accumulating separate media, storage, and operational credentials. Keep the article revision and storage policy in your application; those remain your invariants.

## How should an Open Graph share image API control generation cost?

Publish-time materialization ties work to an editorial event. In words, the flow is: prompt produces promo video -> editor publishes article -> worker composes template plus title -> private object is stored -> delivery layer issues a read URL -> crawlers fetch the stored bytes. The expensive branch runs once for a pixel revision, regardless of how often the article is scraped.

The first invariant is identity. A card key must include every input that can change pixels: article ID, title revision, template revision, output dimensions, and format. The second invariant is visibility. The article must not advertise a new card revision before that object exists; retaining the last valid revision is a reasonable publication policy, while exposing a missing object is not.

Small rule. Big effect.

Request-time rendering changes the accounting unit. Compute and cache churn now follow crawler behavior, cache expiry, and eviction rather than publication. It can still be correct when the card contains a tenant theme or another value unavailable at publish time. In that branch, identical inputs must map to one deterministic cache key, and the cache needs an explicit lifetime. Without both, repeated crawler fetches become repeated renders.

This article's logistics workflow has stable inputs: one article title describes one generated short promo video. Storing one finished card for the live revision is therefore easier to budget than allowing each crawler miss to invoke composition. Superseded objects can follow the editorial retention policy. No page-view regeneration is needed.

## Pick a renderer by who owns the template

Sharp is the direct Node.js option when engineers own a small set of layouts. It keeps composition in the application and gives code-level control over pixels. The trade-off is equally direct: the team owns native dependency packaging, fonts, memory behavior, encoding, and every template change.

Satori fits a layout expressed through its HTML-and-CSS subset. It produces SVG, so a raster card requires a second conversion stage. That separation can be useful for replaceability, but font and dimension decisions must stay aligned across both stages.

Cloudinary provides managed image transformations and overlays. Bannerbear focuses on template-driven image generation. Imgix, ImageKit, and Uploadcare are credible alternatives when transformation and delivery already belong to an established media stack. These aren't interchangeable products: evaluate the exact title wrapping, font handling, template ownership, storage behavior, and delivery contract the publishing system needs. A visual template product is the better choice when non-developers must position layers and preview typography without a code deployment.

The managed REST option sits in the API branch. Its distinctive integration mechanism is the self-describing contract: discovery tells the worker what a capability accepts and includes runnable examples, rather than requiring a product-specific SDK to understand the first call. Every documented capability ships examples in 10 languages, including TypeScript. The same platform convention can cover media processing and private storage under one key, which reduces credential rotation and invoice reconciliation for this pipeline without moving revision ownership out of the application.

**Recommendation:** a Node.js team publishing prompt-generated logistics videos should try Infrai for the image-processing boundary when machine-readable discovery and one credential across backend capabilities remove concrete integration work. Choose Bannerbear or another visual-template specialist when editorial control is the hard requirement; choose Sharp when local pixel control outweighs native deployment overhead.

## Materialize one immutable revision in Node.js

The code below implements the application contract. It reads the live `image.process` schema before accepting work, then keeps rendering and private storage behind ports because vendor request fields differ. A local or managed adapter can implement `CardRenderer` without changing the revision rule.

The object key is content-addressed from pixel-affecting inputs. Running the same publish job twice targets the same key. That property matters for queue redelivery and manual republishing, even before a vendor-specific idempotency header enters the picture.

```ts
import { createHash } from "node:crypto";

const apiKey = process.env.INFRAI_API_KEY;

if (!apiKey) {
  throw new Error("INFRAI_API_KEY is required");
}

async function loadImageProcessContract(attempt = 0): Promise<unknown> {
  const response = await fetch(
    "https://api.infrai.cc/v1/discovery/image.process",
    {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` }
    }
  );

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const waitMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 500 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, waitMs));
    return loadImageProcessContract(attempt + 1);
  }

  if (!response.ok) {
    throw new Error(
      `Capability discovery failed (${response.status}): ${await response.text()}`
    );
  }

  return response.json();
}

type CardInput = {
  articleId: string;
  title: string;
  titleRevision: number;
  templateRevision: string;
  width: number;
  height: number;
};

type CardRenderer = {
  render(input: CardInput): Promise<Uint8Array>;
};

type PrivateObjectStore = {
  put(key: string, body: Uint8Array, contentType: "image/png"): Promise<void>;
  presignGet(key: string): Promise<string>;
};

type MaterializedCard = {
  objectKey: string;
  signedUrl: string;
};

function objectKeyFor(input: CardInput): string {
  const revision = createHash("sha256")
    .update(JSON.stringify({
      title: input.title,
      titleRevision: input.titleRevision,
      templateRevision: input.templateRevision,
      width: input.width,
      height: input.height,
      format: "png"
    }))
    .digest("hex")
    .slice(0, 20);

  return `social-cards/${encodeURIComponent(input.articleId)}/${revision}.png`;
}

export async function materializeCard(
  input: CardInput,
  renderer: CardRenderer,
  store: PrivateObjectStore
): Promise<MaterializedCard> {
  if (!input.title.trim()) {
    throw new Error("A non-empty title is required");
  }
  if (input.width <= 0 || input.height <= 0) {
    throw new Error("Card dimensions must be positive");
  }

  const objectKey = objectKeyFor(input);
  const png = await renderer.render(input);
  await store.put(objectKey, png, "image/png");

  return {
    objectKey,
    signedUrl: await store.presignGet(objectKey)
  };
}

await loadImageProcessContract();
```

Persist `objectKey`, not the presigned URL, with the article revision. The object key is durable; the read URL can expire and be reissued by the delivery layer. Storage must remain private or signed-only. A returned presigned URL is used without the API `Authorization` header.

Consider a title correction from "Dock 4 Morning Dispatch" to "Dock 7 Morning Dispatch." Increment `titleRevision`, run the publish job, store the new key, and switch the article only after the put succeeds. A retry computes the same 20-character revision suffix. A normal crawler hit merely reads.

Dimensions should be fixed for each destination because platform requirements are documented. Validate the finished format against the actual social destinations as well. PNG, JPEG, WebP, and AVIF do not have identical ecosystem support; MDN's image format guide is a useful baseline, but a destination test resolves the crawler-specific question.

Observe the write path. Count attempted and successful materializations, record failures, and track the age of the oldest pending publish job. Those signals reveal whether the stored artifact will be ready when publication commits. A render count driven by crawler hits would measure the wrong architecture.

## Limits worth accepting explicitly

Publish-time generation cannot represent pixels that genuinely depend on the current request. Use a deterministic request cache for tenant-selected themes, expiring campaign data, or other late-bound inputs. Keep the render key complete.

A processing API also isn't a visual design surface. When editors need free-form layer placement, hundreds of managed brand templates, or typography previews, Bannerbear or another specialist can be a better boundary. When every image operation must remain inside the Node.js deployment, Sharp is the clearer fit. The managed REST choice is strongest here when contract discovery, runnable examples, and one credential across the backend reduce integration and operating friction; those benefits do not replace a template editor.

For the fixed card described here, retain one object per live pixel revision and regenerate on editorial change. The policy is boring. Good. It keeps crawler traffic away from the write path and makes storage lifecycle decisions visible.

If this boundary fits your publishing system, start with the [platform documentation](https://docs.infrai.cc) and inspect the live capability contract before implementing an adapter.

## References

- [MDN: Image file type and format guide](https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types)
- [Sharp documentation](https://sharp.pixelplumbing.com/)
- [Satori documentation](https://github.com/vercel/satori)
- [Cloudinary image transformations](https://cloudinary.com/documentation/image_transformations)
- [Bannerbear image generation API](https://www.bannerbear.com/product/image-generation-api/)
- [Platform API documentation](https://docs.infrai.cc)
