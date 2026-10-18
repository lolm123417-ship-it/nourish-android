# Meal-photo analysis: setup and status

## What this adds
- **Server** (`artifacts/api-server`): `POST /api/meal-photo/analyze` sends a
  compressed photo to Google Gemini, validates the structured answer, and returns
  foods, estimated grams, calories, protein, carbs, fat, fiber (when estimable),
  confidence, visible evidence, assumptions, and warnings.
- **App** (`artifacts/nourish`): "Estimate from a food photo" on the Log food
  screen opens `app/photo-analyze.tsx`. The user takes or chooses a photo, reviews
  every item, edits weights, removes or adds items, and confirms. Nothing reaches
  the diary without confirmation.
- **Nutrition logic** (`lib/photoAnalysis.ts`): weight edits scale the original
  estimate proportionally, totals are summed in code, and entries are tagged
  `ai-photo` or `manual` so their origin stays visible.

## Verification status
Tested here:
- Client logic: 21 tests pass (`pnpm test` in `artifacts/nourish`), including
  persistence validation of saved entries.
- Server logic: 8 tests pass (`node --experimental-strip-types --test src/lib/mealPhotoAnalysis.test.mjs`
  in `artifacts/api-server`), covering response validation, safety blocks,
  truncation, request construction, key safety, and rate limiting.
- Strict TypeScript passes on the pure modules. JSX files parse. The server bundles.

Not tested (no network or Android toolchain in the build environment):
- A live call to Gemini. The model ID and request format must be checked once.
- The Express route running on a real host.
- Any APK build or run on a device.

## Setup
1. **Get a Gemini key** in Google AI Studio. Check that the model ID in
   `GEMINI_MODEL` (default `gemini-3.6-flash`) is listed there. Model names change
   often, so confirm before deploying.
2. **Deploy the server** (`artifacts/api-server`) to a Node 20+ host you choose.
   Build with `pnpm --filter @workspace/api-server run build`, then run
   `node dist/index.mjs`. Set these environment variables on the host:
   - `PORT` (required by the server)
   - `GEMINI_API_KEY` (required; keep it on the server only)
   - `GEMINI_MODEL` (optional)
   - `MEAL_PHOTO_HOURLY_LIMIT` (optional, default 20 analyses per client per hour)
3. **Point the app at the server** by setting `EXPO_PUBLIC_API_BASE_URL` to the
   server's HTTPS origin (for example `https://your-server.example.com`, with no
   trailing slash) in the environment used to build the app.
4. **Build the APK.** Options include EAS Build (`npx eas-cli build -p android --profile preview`,
   needs an Expo account and may have usage limits) or a local build (`npx expo prebuild`,
   then Gradle, needs JDK 17+ and the Android SDK). Neither was run here.

## Privacy and cost
- The server does not store photos. It does forward each photo to Google.
- Google's free tier may use submitted content to improve its products. Use a key
  and billing setup whose data terms you accept.
- Each analysis costs Gemini quota. The per-client limit above is in-memory and
  resets when the server restarts. Set `app.set("trust proxy", ...)` if the server
  sits behind a proxy, or every user will share one limit.

## Known limitations
- Barcode lookup (Open Food Facts) works as before. Photo-of-package fallback
  is not implemented.
- Fiber is stored with photo entries but not yet shown in the diary.
- Estimates are approximate. The review screen flags calorie and macro mismatches,
  but it cannot verify portions.
