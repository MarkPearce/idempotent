# Database Guide

Comprehensive database documentation for PostgreSQL applications using Supabase with idempotent migration patterns.

## Overview

This database guide provides:

- Domain-driven data model patterns for scalable applications
- Idempotent migration system for reliable deployments
- Schema versioning and change tracking
- Helper functions for common database operations

## Data Model Principles

### Design Patterns

- All core objects use UUIDs as primary identifiers
- Entities are designed as reusable models referenced by ID
- System supports both real-time and asynchronous operations
- Clear separation of concerns between different data domains

### Best Practices

- Use consistent naming conventions across all database objects
- Implement proper foreign key relationships for data integrity
- Design for scalability with appropriate indexing strategies
- Consider data access patterns when designing table structures

## Migration System

### Migration Naming Convention

**CRITICAL**: Migrations MUST follow the format: `YYYYMMDDHHMMSS_descriptive_name.sql`

```bash
# Examples
20250703140000_fix_user_profiles_updated_at.sql
20250703140001_add_schema_versions_unique_constraint.sql
20250703140002_enhance_user_tracking.sql
```

**Rules**:
- Always use current or future timestamps
- Increment time by at least 1 second for multiple migrations on same day
- Sequential ordering is critical for production deployments

### Helper Functions

The system provides idempotent helper functions:

#### create_table_if_not_exists

```sql
SELECT create_table_if_not_exists(
  p_table_name := 'user_preferences',
  p_creation_sql := $create_sql$
    CREATE TABLE user_preferences (
      id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
      user_id UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
      theme TEXT NOT NULL DEFAULT 'light',
      created_at TIMESTAMPTZ DEFAULT NOW()
    )
  $create_sql$
);
```

#### add_column_if_not_exists

```sql
SELECT add_column_if_not_exists(
  p_table_name := 'users', 
  p_column_name := 'last_login', 
  p_column_type := 'TIMESTAMPTZ'
);
```

#### create_index_if_not_exists

```sql
-- Single column index
SELECT create_index_if_not_exists(
  'users_email_idx', 
  'users', 
  'email'
);

-- Multi-column composite index
SELECT create_index_if_not_exists(
  'user_activities_user_date_idx', 
  'user_activities', 
  'user_id, created_at'
);

-- Partial index with WHERE clause
SELECT create_index_if_not_exists(
  'active_sessions_idx', 
  'sessions', 
  'created_at', 
  '', 
  'status = ''active'''
);
```

## Migration Template

**Location**: `supabase/migrations/migration_template.sql`

```sql
-- Migration: [DESCRIPTION]
-- Created: [DATE]
-- Filename: YYYYMMDDHHMMSS_descriptive_name.sql

BEGIN;

-- Example: Adding a new table
SELECT create_table_if_not_exists(
  p_table_name := 'new_table',
  p_creation_sql := $create_sql$
    CREATE TABLE new_table (
      id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
      name TEXT NOT NULL,
      created_at TIMESTAMPTZ DEFAULT NOW()
    )
  $create_sql$
);

-- Example: Adding a column
SELECT add_column_if_not_exists(
  p_table_name := 'users', 
  p_column_name := 'last_login', 
  p_column_type := 'TIMESTAMPTZ'
);

-- Example: Creating an index
SELECT create_index_if_not_exists(
  'users_email_idx', 
  'users', 
  'email'
);

-- Record this migration
INSERT INTO schema_versions (version, description)
VALUES ('[MIGRATION_VERSION]', '[MIGRATION_DESCRIPTION]');

COMMIT;
```

## Idempotent Patterns

### Creating Tables with DO Blocks

```sql
DO $$
BEGIN
  IF NOT EXISTS (
    SELECT FROM information_schema.tables 
    WHERE table_schema = 'public' AND table_name = 'users'
  ) THEN
    CREATE TABLE users (
      id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
      email TEXT NOT NULL UNIQUE,
      name TEXT NOT NULL,
      created_at TIMESTAMPTZ DEFAULT NOW()
    );
  END IF;
END $$;
```

### Adding Foreign Keys

```sql
DO $$ 
BEGIN
  IF NOT EXISTS (
    SELECT 1 FROM information_schema.table_constraints 
    WHERE constraint_name = 'profiles_user_id_fkey'
  ) THEN
    ALTER TABLE profiles
    ADD CONSTRAINT profiles_user_id_fkey
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE;
  END IF;
END $$;
```

### Creating Enum Types

```sql
DO $$
BEGIN
  IF NOT EXISTS (SELECT 1 FROM pg_type WHERE typname = 'user_role') THEN
    CREATE TYPE user_role AS ENUM ('admin', 'user', 'guest');
  END IF;
END$$;
```

### Inserting Data Safely

```sql
DO $$
BEGIN
  IF NOT EXISTS (SELECT 1 FROM users WHERE email = 'admin@example.com') THEN
    INSERT INTO users (email, name, role)
    VALUES ('admin@example.com', 'Admin User', 'admin');
  END IF;
END$$;
```

## Schema Versioning

### Schema Versions Table

```sql
CREATE TABLE schema_versions (
  id SERIAL PRIMARY KEY,
  version TEXT NOT NULL,
  applied_at TIMESTAMPTZ DEFAULT NOW(),
  description TEXT
);
```

### Recording Migrations

Each migration should record its application:

```sql
INSERT INTO schema_versions (version, description)
VALUES ('20250519000001_add_user_metadata', 'Adds user metadata fields');
```

### Querying Schema Status

```sql
-- Check applied migrations
SELECT version, applied_at, description
FROM schema_versions
ORDER BY applied_at;

-- Check if specific migration applied
SELECT EXISTS (
  SELECT 1 FROM schema_versions
  WHERE version = '20250519000001_add_user_metadata'
);
```

## Development Workflow

### Creating New Migrations

```bash
# 1. Copy template with timestamp
cp supabase/migrations/migration_template.sql supabase/migrations/$(date +%Y%m%d%H%M%S)_your_migration_name.sql

# 2. Edit the migration file
# 3. Test locally
supabase db reset

# 4. Apply migration
supabase db push

# 5. Generate types (if using TypeScript)
supabase gen types typescript --local > src/types/database.ts
```

### Local Development Commands

```bash
# Start Supabase locally
supabase start

# Apply migrations
supabase db push

# Reset database (applies all migrations)
supabase db reset

# Connect to database
supabase db shell

# View migration status
supabase migration list
```

## Best Practices

### Migration Development

1. **Always Use Transactions**: Wrap migrations in `BEGIN`/`COMMIT` blocks
2. **Use Helper Functions**: Leverage provided helper functions for common operations
3. **Test Idempotency**: Verify migrations can be run multiple times safely
4. **Document Changes**: Include clear comments and descriptions
5. **Record Versions**: Always insert a record into `schema_versions`

### Naming Conventions

- **Files**: `YYYYMMDDHHMMSS_descriptive_name.sql`
- **Tables**: `snake_case` with descriptive names
- **Columns**: `snake_case` following PostgreSQL conventions
- **Indexes**: `tablename_columnname_idx`
- **Constraints**: `tablename_columnname_constraint_type`

### Data Types

- **Primary Keys**: Use `UUID` with `uuid_generate_v4()`
- **Timestamps**: Use `TIMESTAMPTZ` for timezone awareness
- **Text**: Use `TEXT` instead of `VARCHAR` unless length limits required
- **Booleans**: Use `BOOLEAN` with explicit defaults
- **JSON**: Use `JSONB` for better performance

## Troubleshooting

### Common Issues

**Migration Fails to Apply**:
1. Check for syntax errors in SQL
2. Verify dependencies - ensure referenced tables/columns exist
3. Check for naming conflicts
4. Review transaction boundaries

**Idempotency Issues**:
1. Test multiple runs locally
2. Check all IF EXISTS/NOT EXISTS conditions
3. Verify helper functions are available

### Debug Commands

```bash
# Check migration status
supabase migration list

# View database logs
supabase logs db

# Check schema versions
psql -c "SELECT * FROM schema_versions ORDER BY applied_at;"
```

### Recovery Procedures

**Failed Migration**:
1. Identify issue from error logs
2. Fix migration file
3. Reset database: `supabase db reset`
4. Reapply migrations: `supabase db push`

## Advanced Patterns

### Complex Schema Changes

```sql
DO $$
BEGIN
  -- Create new table with desired structure
  IF NOT EXISTS (
    SELECT FROM information_schema.tables 
    WHERE table_schema = 'public' AND table_name = 'users_new'
  ) THEN
    CREATE TABLE users_new (
      id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
      full_name TEXT NOT NULL,
      email TEXT NOT NULL UNIQUE,
      created_at TIMESTAMPTZ DEFAULT NOW()
    );
    
    -- Migrate data from old table
    INSERT INTO users_new (id, full_name, email, created_at)
    SELECT id, CONCAT(first_name, ' ', last_name), email, created_at
    FROM users;
    
    -- Rename tables
    ALTER TABLE users RENAME TO users_old;
    ALTER TABLE users_new RENAME TO users;
  END IF;
END $$;
```

### Data Seeding

```sql
DO $$
BEGIN
  -- Insert default roles
  INSERT INTO roles (name, description)
  SELECT 'admin', 'Administrator role'
  WHERE NOT EXISTS (SELECT 1 FROM roles WHERE name = 'admin');
  
  INSERT INTO roles (name, description)
  SELECT 'user', 'Standard user role'
  WHERE NOT EXISTS (SELECT 1 FROM roles WHERE name = 'user');
END $$;
```

### Performance Optimization

```sql
-- Create partial indexes for common queries
SELECT create_index_if_not_exists(
  'active_users_idx',
  'users',
  'last_login',
  '',
  'status = ''active'' AND last_login > NOW() - INTERVAL ''30 days'''
);

-- Create composite indexes for complex queries
SELECT create_index_if_not_exists(
  'user_activity_lookup_idx',
  'user_activities',
  'user_id, activity_type, created_at DESC'
);
```

## Schema Documentation

Schema documentation should be maintained alongside your codebase:

- **Note**: Documentation files are for reference - database schema is the source of truth
- **Structure**: Organize by domain or feature area
- **Purpose**: Provides human-readable documentation of data models
- **Format**: Use consistent format (YAML, Markdown, or JSON) across your project

## Integration Patterns

### Service Layer Integration

```typescript
// Example service pattern for database operations
export class BaseService {
  protected supabase: SupabaseClient;
  
  constructor(supabase: SupabaseClient) {
    this.supabase = supabase;
  }
  
  protected async executeQuery<T>(
    query: PostgrestQueryBuilder<any, any, any>
  ): Promise<T[]> {
    const { data, error } = await query;
    if (error) throw new Error(`Database error: ${error.message}`);
    return data || [];
  }
}
```

### Error Handling

```typescript
// Consistent error handling for database operations
export class DatabaseError extends Error {
  constructor(
    message: string,
    public code?: string,
    public details?: any
  ) {
    super(message);
    this.name = 'DatabaseError';
  }
}
```

### Type Safety

```typescript
// Use generated types for type safety
import { Database } from './types/database';

type User = Database['public']['Tables']['users']['Row'];
type UserInsert = Database['public']['Tables']['users']['Insert'];
type UserUpdate = Database['public']['Tables']['users']['Update'];
```

This database guide provides essential patterns for working with PostgreSQL databases using Supabase. Always use the migration template and follow idempotent patterns for reliable database evolution.
