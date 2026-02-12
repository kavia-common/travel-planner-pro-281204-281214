# Travel Planner Database Schema (PostgreSQL)

This document describes the intended schema for the Travel Planner app.

**Important:** Per container rules, apply SQL statements using `psql -c "..."` **one statement at a time** using the connection command in `db_connection.txt`.

## Connection

`db_connection.txt` contains a ready-to-use command, e.g.:

- `psql postgresql://appuser:dbuser123@localhost:5000/myapp`

## Tables overview

- `users`: application users
- `trips`: a trip owned by a user
- `trip_destinations`: destinations (city/country) attached to a trip (with order and optional dates)
- `itinerary_days`: per-trip day entries (date + title/summary)
- `itinerary_items`: scheduled items for a specific itinerary day (optionally linked to accommodation/activity)
- `accommodations`: lodging entries tied to a trip (optionally tied to a destination)
- `activities`: activity entries tied to a trip (optionally tied to a destination and/or itinerary day)
- `notes`: free-form notes tied to a trip (optionally tied to a destination and/or day)

## DDL (apply one statement at a time)

### Extensions

```sql
CREATE EXTENSION IF NOT EXISTS pgcrypto;
```

### users

```sql
CREATE TABLE IF NOT EXISTS users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email TEXT NOT NULL UNIQUE,
  full_name TEXT NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### trips

```sql
CREATE TABLE IF NOT EXISTS trips (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  name TEXT NOT NULL,
  start_date DATE,
  end_date DATE,
  home_timezone TEXT,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

Indexes:

```sql
CREATE INDEX IF NOT EXISTS idx_trips_user_id ON trips(user_id);
```

### trip_destinations

```sql
CREATE TABLE IF NOT EXISTS trip_destinations (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  trip_id UUID NOT NULL REFERENCES trips(id) ON DELETE CASCADE,
  name TEXT NOT NULL,
  country TEXT,
  start_date DATE,
  end_date DATE,
  sort_order INTEGER NOT NULL DEFAULT 0,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

Indexes:

```sql
CREATE INDEX IF NOT EXISTS idx_trip_destinations_trip_id ON trip_destinations(trip_id);
```

### itinerary_days

```sql
CREATE TABLE IF NOT EXISTS itinerary_days (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  trip_id UUID NOT NULL REFERENCES trips(id) ON DELETE CASCADE,
  day_date DATE NOT NULL,
  title TEXT,
  summary TEXT,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE(trip_id, day_date)
);
```

Indexes:

```sql
CREATE INDEX IF NOT EXISTS idx_itinerary_days_trip_id ON itinerary_days(trip_id);
```

### accommodations

```sql
CREATE TABLE IF NOT EXISTS accommodations (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  trip_id UUID NOT NULL REFERENCES trips(id) ON DELETE CASCADE,
  destination_id UUID REFERENCES trip_destinations(id) ON DELETE SET NULL,
  name TEXT NOT NULL,
  address TEXT,
  check_in TIMESTAMPTZ,
  check_out TIMESTAMPTZ,
  confirmation_number TEXT,
  booking_url TEXT,
  phone TEXT,
  notes TEXT,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

Indexes:

```sql
CREATE INDEX IF NOT EXISTS idx_accommodations_trip_id ON accommodations(trip_id);
```

### activities

```sql
CREATE TABLE IF NOT EXISTS activities (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  trip_id UUID NOT NULL REFERENCES trips(id) ON DELETE CASCADE,
  destination_id UUID REFERENCES trip_destinations(id) ON DELETE SET NULL,
  day_id UUID REFERENCES itinerary_days(id) ON DELETE SET NULL,
  name TEXT NOT NULL,
  start_time TIMESTAMPTZ,
  end_time TIMESTAMPTZ,
  location TEXT,
  booking_url TEXT,
  cost NUMERIC(12,2),
  notes TEXT,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

Indexes:

```sql
CREATE INDEX IF NOT EXISTS idx_activities_trip_id ON activities(trip_id);
CREATE INDEX IF NOT EXISTS idx_activities_day_id ON activities(day_id);
```

### itinerary_items

```sql
CREATE TABLE IF NOT EXISTS itinerary_items (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  day_id UUID NOT NULL REFERENCES itinerary_days(id) ON DELETE CASCADE,
  item_type TEXT NOT NULL, -- 'activity', 'accommodation', 'custom'
  title TEXT NOT NULL,
  start_time TIMESTAMPTZ,
  end_time TIMESTAMPTZ,
  activity_id UUID REFERENCES activities(id) ON DELETE SET NULL,
  accommodation_id UUID REFERENCES accommodations(id) ON DELETE SET NULL,
  details TEXT,
  sort_order INTEGER NOT NULL DEFAULT 0,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

Indexes:

```sql
CREATE INDEX IF NOT EXISTS idx_itinerary_items_day_id ON itinerary_items(day_id);
```

### notes

```sql
CREATE TABLE IF NOT EXISTS notes (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  trip_id UUID NOT NULL REFERENCES trips(id) ON DELETE CASCADE,
  destination_id UUID REFERENCES trip_destinations(id) ON DELETE SET NULL,
  day_id UUID REFERENCES itinerary_days(id) ON DELETE SET NULL,
  title TEXT,
  content TEXT NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

Indexes:

```sql
CREATE INDEX IF NOT EXISTS idx_notes_trip_id ON notes(trip_id);
```
