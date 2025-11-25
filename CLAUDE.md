# C2C Web Application - Technical Documentation

> **App Purpose**: Cafe discovery and rating platform for remote workers and coffee enthusiasts

## What is C2C?

C2C (Cafe-to-Cafe) helps users discover and rate cafes based on work-friendly criteria. Users can search for nearby cafes, view detailed ratings across 6 categories, and submit their own ratings to help others find the perfect workspace.

### Key Features (Current Implementation)
- **Interactive Map View**: Mapbox-powered map showing nearby cafes within 2 miles
- **Location-Based Search**: GPS-powered cafe discovery + name search
- **6-Category Rating System**: Coffee, Vibe, WiFi, Outlets, Seating, Noise
- **Interactive Star Ratings**: Hover-responsive UI with half-star precision
- **Expandable Cafe Cards**: Click to see detailed ratings with category icons
- **Real-time Updates**: Database-backed ratings aggregation

---

## Technology Stack

### Frontend
- **Framework**: Next.js 15 (App Router, React 19)
- **Language**: TypeScript 5.3+
- **UI**: Tailwind CSS with custom amber/beige theme
- **Maps**: Mapbox GL JS (react-map-gl)
- **Icons**: Custom pixel-art PNG assets for rating categories

### Backend & Database
- **Database**: Supabase (PostgreSQL with PostGIS extension)
- **Tables**:
  - `cafes`: Cafe locations, metadata, geospatial data
  - `ratings`: User ratings across 6 categories
  - `profiles`: User profiles with vibe preferences (lock-in, network, chill)
- **API**: Geoapify Places API for cafe discovery
- **Hosting**: Vercel (Next.js) + Supabase Cloud

---

## Database Schema (Supabase/PostgreSQL)

### Core Tables

**`cafes`**
```sql
- id (uuid, primary key)
- geoapify_place_id (text, unique)
- osm_id (text)
- name (text)
- address (text)
- location (geography point) -- PostGIS for geospatial queries
- phone, website (text, nullable)
- user_photos (text[])
- verified_hours (jsonb)
- first_discovered_at, last_synced_at (timestamp)
- created_at, updated_at (timestamp)
```

**`ratings`**
```sql
- id (uuid, primary key)
- cafe_id (uuid, foreign key → cafes)
- user_id (uuid, foreign key → profiles)
- coffee_rating (numeric 0-5, nullable)
- vibe_rating (numeric 0-5, nullable)
- wifi_rating (numeric 0-5, nullable)
- outlets_rating (numeric 0-5, nullable)
- seating_rating (numeric 0-5, nullable)
- noise_rating (numeric 0-5, nullable)
- overall_rating (numeric 0-5, REQUIRED)
- comment (text, nullable)
- photos (text[])
- created_at, updated_at (timestamp)
```

**`profiles`**
```sql
- id (uuid, primary key, linked to auth.users)
- username (text, unique)
- is_onboarded (boolean)
- metadata (jsonb) -- stores: { vibe: 'lock-in' | 'network' | 'chill', preferences, stats }
- created_at, updated_at (timestamp)
```

### Materialized View
**`cafe_stats`** - Pre-aggregated rating averages for performance
- Refreshed automatically on new ratings
- Provides avg_coffee, avg_vibe, avg_wifi, avg_outlets, avg_seating, avg_noise, avg_overall

### Database Functions
- `get_nearby_cafes(lat, lng, radius_meters, min_rating, limit)`: Geospatial search with rating filters
- `refresh_cafe_stats()`: Refresh materialized view
- `get_user_profile_with_stats(user_id)`: Get profile + rating statistics

---

## UI/UX Design

### Current Interface (Map View)

**Left Panel**: Cafe list with search
- Search bar (by name)
- "Nearby (2mi)" button - GPS-based search
- Results count badge
- Scrollable cafe cards showing:
  - Ranking (#1, #2, etc.)
  - Cafe name, distance, address
  - Overall star rating + review count
  - **Expandable section (click to expand)**

**Expanded Cafe View** (When clicked):
- Header: "Rate this cafe:"
- 6 rating categories with pixel-art icons + interactive stars:
  - ☕ **Coffee** (`/assets/coffee.png`)
  - 🎵 **Vibe** (`/assets/vibes.png`)
  - 📶 **WiFi** (`/assets/wifi.png`)
  - 🔌 **Outlets** (`/assets/plugs.png`)
  - 💺 **Seating** (`/assets/seats.png`)
  - 🔊 **Noise** (`/assets/noise.png`)

**Interactive Star Rating Behavior**:
- **Default**: 5 empty stars (`zero_star.png`) when rating = 0
- **Hover left half of star**: Shows half star (`half_star.png`)
- **Hover right half of star**: Shows full star (`full_star.png`)
- **Smooth gliding**: Stars update reactively as mouse moves horizontally
- **Click**: Sets rating (0.5 increments, ready for API submission)
- **Display**: Numerical rating shown (e.g., "3.5")

**Map View** (Right side):
- Mapbox custom style
- Cafe markers with custom coffee cup icon (`cafe-icon.png`)
- Selected cafe highlighted (beige pin)
- User location (blue pulsing dot)
- Geolocate + zoom controls
- **Interactions**:
  - Click marker → centers map & scrolls cafe into view in left panel
  - Click cafe card → centers map on that cafe

**Panel Controls**:
- Collapse/expand button (ChevronLeft/Right)
- Panel width: 384px (w-96) when expanded, 0 when collapsed
- Map resizes automatically

### Color Scheme & Style
- **Primary**: Amber/beige tones (`bg-amber-50`, `bg-amber-100`, `border-amber-900`)
- **Text**: Dark amber (`text-amber-900`, `text-amber-800`)
- **Buttons**: Amber-700 background with hover states
- **Selected**: Amber-100 background, amber-700 border
- **Style**: Pixel-art aesthetic with sharp borders (no rounded corners on assets)
- **Font**: System font with pixel-image class for PNG assets

---

## API Routes

### `GET /api/cafes/nearby?lat={lat}&lng={lng}`
Returns cafes within 2-mile radius (3219 meters), sorted by distance.

**Flow**:
1. Query Supabase for existing cafes in radius
2. If < 10 cafes in DB → Fetch from Geoapify API
3. Store new cafes in database
4. Refresh materialized view
5. Return all cafes with aggregated ratings

**Response**:
```json
{
  "success": true,
  "count": 25,
  "cafes": [
    {
      "id": "51daa8dc8eb5a90c40593829dce17e954940f00103f90105e4e90100000000c0020192031654617465277320436f666665652026204b69746368656e",
      "name": "Fate's Coffee & Kitchen",
      "location": { "lat": 37.7749, "lng": -122.4194 },
      "address": "123 Main St, San Francisco, CA",
      "placeId": "...",
      "ratings": {
        "coffee": 0,
        "vibe": 0,
        "wifi": 0,
        "outlets": 0,
        "seating": 0,
        "noise": 0,
        "overall": 0
      },
      "totalReviews": 0,
      "photos": [],
      "distance": 450,
      "website": "https://...",
      "phone": "+1234567890"
    }
  ],
  "searchCenter": { "lat": 37.7749, "lng": -122.4194 },
  "radiusMeters": 3219,
  "radiusMiles": 2,
  "source": "database", // or "geoapify+database"
  "cacheHit": true
}
```

### `GET /api/cafes/search?q={query}&lat={lat}&lng={lng}`
Search cafes by name within radius.

---

## Component Architecture

### Key Components

**`components/map/MapView.tsx`**
- Main map interface
- Manages cafe list state, selected cafe, expanded cafe
- Handles geolocation, search, cafe clicks
- Renders Mapbox map + markers
- Left panel with cafe cards

**`components/ui/StarRating.tsx`** ⭐
- Reusable interactive star rating component
- Props:
  - `rating` (0-5): Current rating value
  - `maxStars` (default 5): Number of stars
  - `size` (pixels): Star image size
  - `interactive` (boolean): Enable hover/click
  - `onChange` (callback): Rating change handler
  - `showNumber` (boolean): Display numerical rating
- Features:
  - Precise mouse tracking for half-star detection
  - Hover preview vs. actual rating
  - Click to set rating
  - Accessible with keyboard support (future)

**`lib/supabase.ts`**
- Supabase client configuration
- TypeScript types for database tables

**`types/cafe.ts`**
- Core TypeScript interfaces:
  - `Cafe`: Full cafe data
  - `CafeRatings`: 6 rating categories + overall
  - `Coordinate`: lat/lng

---

## Environment Variables

Required in `.env.local`:
```bash
NEXT_PUBLIC_MAPBOX_TOKEN=pk.eyJ1...
GEOAPIFY_API_KEY=...
NEXT_PUBLIC_SUPABASE_URL=https://....supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=eyJhbGciOiJ...
SUPABASE_SERVICE_ROLE_KEY=eyJhbGciOiJ...
```

---

## Development Tools & Workflow

### AI Tools in Use
- **Claude Code**: Architecture decisions, debugging, code reviews, complex refactoring
- **Cursor IDE**: Day-to-day coding with AI pair programming (Cmd+K for inline generation)
- **v0.dev**: Rapid UI component prototyping (used for initial designs)

### Development Commands
```bash
npm run dev          # Start Next.js dev server (port 3000)
npm run build        # Production build
npm run lint         # ESLint check
```

### Code Style
- TypeScript strict mode enabled
- Tailwind CSS for all styling (no CSS modules)
- Component files use `'use client'` directive when needed (map interactivity)
- PNG assets use `unoptimized` flag + `pixel-image` class

---

## Future Features (Roadmap)

### Phase 1: Rating Submission
- API endpoint: `POST /api/ratings`
- Form validation (overall rating required, others optional)
- Photo upload to Supabase Storage
- Optimistic UI updates

### Phase 2: User Authentication
- Supabase Auth (Google OAuth, Email)
- Protected routes for rating submission
- User profile page
- Onboarding flow (select vibe preference)

### Phase 3: Social Features
- View user's rating history
- Follow other users
- Cafe favorites/bookmarks
- Activity feed

### Phase 4: AI-Powered Recommendations
- Natural language search: "quiet cafe with good wifi"
- Personalized recommendations based on vibe preference
- Semantic search using embeddings
- Auto-suggest review text

---

## Key Implementation Details

### Geospatial Queries
- PostGIS `geography` type for precise distance calculations
- `ST_DWithin` for radius searches
- Distance returned in meters, converted to miles in frontend

### Rating Aggregation Strategy
- Materialized view (`cafe_stats`) for performance
- Refreshed on each new rating insertion
- Avoids expensive aggregations on every query
- Displays 0.0 for categories with no ratings

### Interactive Star Rating Algorithm
```typescript
// In StarRating.tsx
const handleMouseMove = (e: MouseEvent) => {
  const rect = containerRef.current.getBoundingClientRect();
  const x = e.clientX - rect.left;
  const starWidth = rect.width / maxStars;
  const starIndex = Math.floor(x / starWidth);
  const positionInStar = (x % starWidth) / starWidth;

  // Left half = 0.5, Right half = 1.0
  const rating = positionInStar < 0.5
    ? starIndex + 0.5
    : starIndex + 1;
}
```

### Search Performance Optimization
- Cache cafes in database on first search
- Check cache before hitting Geoapify API
- Only fetch if < 10 cafes in DB (MIN_RESULTS_THRESHOLD)
- Saves API costs and improves speed

---

## Troubleshooting

### Common Issues

**Supabase Connection Errors**
- Verify `.env.local` has correct credentials
- Check Supabase project is running
- Ensure PostGIS extension is enabled
- Test with: `curl https://YOUR_PROJECT.supabase.co/rest/v1/`

**Map Not Rendering**
- Verify Mapbox token is valid
- Check browser console for errors
- Ensure internet connection for tile loading

**Stars Not Interactive**
- Verify `interactive={true}` prop passed
- Check browser console for JavaScript errors
- Ensure PNG assets exist in `/public/assets/`

**No Cafes Found**
- Check GPS permissions granted
- Verify location is within supported areas
- Check Geoapify API key is valid
- Look for API errors in server console

---

## Contact & Contribution

For questions or suggestions, open an issue on GitHub or contact the development team.

**Tech Stack Summary**: Next.js 15 + TypeScript + Supabase + Mapbox + Tailwind CSS
**Status**: Active development (MVP complete, adding rating submission next)
