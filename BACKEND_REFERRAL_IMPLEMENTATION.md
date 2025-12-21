# Backend Referral System Implementation Guide

This document outlines all the backend changes needed to support the referral system in TapinX.

## PART 1: Update Signup Route to Accept Referral Data

### Location: `routes/users.js` or `server.js` (signup endpoint)

Update the signup route to accept and save referral data:

```javascript
// POST /api/signup
router.post('/signup', async (req, res) => {
    try {
        const { phone, name, age, referred_by, referred_opportunity_id } = req.body;
        
        // Validate required fields
        if (!phone) {
            return res.status(400).json({ success: false, error: 'Phone is required' });
        }
        
        // Check if user already exists
        const existingUser = await client.query(
            'SELECT * FROM users WHERE phone = $1',
            [phone]
        );
        
        let user;
        
        if (existingUser.rows.length > 0) {
            // Update existing user
            user = existingUser.rows[0];
            
            // Update referral data if provided
            if (referred_by !== undefined || referred_opportunity_id !== undefined) {
                await client.query(
                    `UPDATE users 
                     SET referred_by = COALESCE($1, referred_by),
                         referred_opportunity_id = COALESCE($2, referred_opportunity_id),
                         updated_at = NOW()
                     WHERE id = $3`,
                    [
                        referred_by ? parseInt(referred_by) : null,
                        referred_opportunity_id ? parseInt(referred_opportunity_id) : null,
                        user.id
                    ]
                );
            }
        } else {
            // Create new user
            const result = await client.query(
                `INSERT INTO users (name, phone, age, referred_by, referred_opportunity_id, created_at, updated_at)
                 VALUES ($1, $2, $3, $4, $5, NOW(), NOW())
                 RETURNING *`,
                [
                    name || 'User',
                    phone,
                    age ? parseInt(age) : null,
                    referred_by ? parseInt(referred_by) : null,
                    referred_opportunity_id ? parseInt(referred_opportunity_id) : null
                ]
            );
            
            user = result.rows[0];
        }
        
        res.json({
            success: true,
            user: {
                id: user.id,
                name: user.name,
                phone: user.phone,
                age: user.age,
                referred_by: user.referred_by,
                referred_opportunity_id: user.referred_opportunity_id
            }
        });
        
    } catch (error) {
        console.error('[Signup] Error:', error);
        res.status(500).json({ success: false, error: error.message });
    }
});
```

### Database Migration

Make sure the `users` table has these columns:

```sql
ALTER TABLE users 
ADD COLUMN IF NOT EXISTS referred_by INTEGER REFERENCES users(id),
ADD COLUMN IF NOT EXISTS referred_opportunity_id INTEGER REFERENCES asks(id);
```

---

## PART 2: Update Matches Route to Handle 'referred' Status

### Location: `routes/matches.js` or `server.js` (matches respond endpoint)

Update the match response handler to accept 'referred' as a valid status:

```javascript
// POST /api/matches/:id/respond
router.post('/matches/:id/respond', async (req, res) => {
    try {
        const { id } = req.params;
        const { response, user_id } = req.body;
        
        // Validate response
        if (!response || !['yes', 'no', 'declined', 'referred'].includes(response)) {
            return res.status(400).json({ 
                success: false, 
                error: 'Invalid response. Must be: yes, no, declined, or referred' 
            });
        }
        
        // Validate user_id
        if (!user_id) {
            return res.status(400).json({ 
                success: false, 
                error: 'user_id is required' 
            });
        }
        
        await client.query('BEGIN');
        
        // Check if match exists and belongs to user
        const matchCheck = await client.query(
            'SELECT * FROM matches WHERE id = $1 AND matched_user_id = $2',
            [id, user_id]
        );
        
        if (matchCheck.rows.length === 0) {
            await client.query('ROLLBACK');
            return res.status(404).json({ 
                success: false, 
                error: 'Match not found' 
            });
        }
        
        // Map response values
        let status;
        if (response === 'yes') {
            status = 'accepted';
        } else if (response === 'no' || response === 'declined') {
            status = 'declined';
        } else if (response === 'referred') {
            status = 'referred';
        }
        
        // Update match status
        await client.query(
            `UPDATE matches 
             SET status = $1, updated_at = NOW() 
             WHERE id = $2`,
            [status, id]
        );
        
        await client.query('COMMIT');
        
        res.json({ 
            success: true, 
            message: `Match ${status}`,
            status: status
        });
        
    } catch (error) {
        await client.query('ROLLBACK');
        console.error('[Matches] Respond error:', error);
        res.status(500).json({ success: false, error: error.message });
    }
});
```

### Database Migration

Make sure the `matches` table status column accepts 'referred':

```sql
-- Check current status constraint
ALTER TABLE matches 
DROP CONSTRAINT IF EXISTS matches_status_check;

ALTER TABLE matches 
ADD CONSTRAINT matches_status_check 
CHECK (status IN ('pending', 'accepted', 'declined', 'referred', 'expired'));
```

---

## PART 3: Create Archive Endpoint

### Location: `routes/opportunities.js` or create new file

Add a new route to get archived opportunities:

```javascript
/**
 * GET /api/opportunities/:identifier/archive
 * Get archived opportunities (declined + referred) for a user
 */
router.get('/opportunities/:identifier/archive', async (req, res) => {
    try {
        const { identifier } = req.params;
        
        console.log('[Opportunities] Getting archive for:', identifier);
        
        // Resolve user (phone or ID)
        let user;
        if (isNaN(identifier)) {
            // Assume it's a phone number
            const result = await client.query(
                'SELECT * FROM users WHERE phone = $1',
                [identifier]
            );
            user = result.rows[0];
        } else {
            // Assume it's a user ID
            const result = await client.query(
                'SELECT * FROM users WHERE id = $1',
                [parseInt(identifier)]
            );
            user = result.rows[0];
        }
        
        if (!user) {
            return res.json({ 
                success: true, 
                opportunities: [], 
                count: 0 
            });
        }
        
        // Get matches where this user declined or referred
        const result = await client.query(`
            SELECT 
                m.id,
                m.ask_id,
                m.status,
                m.created_at,
                m.updated_at,
                a.title as ask_title,
                a.ask_text,
                a.category,
                u.name as requester_name,
                u.id as requester_id
            FROM matches m
            JOIN asks a ON m.ask_id = a.id
            JOIN users u ON m.requester_id = u.id
            WHERE m.matched_user_id = $1 
            AND m.status IN ('declined', 'referred')
            ORDER BY m.updated_at DESC
        `, [user.id]);
        
        const opportunities = result.rows.map(m => ({
            id: m.id,
            ask_id: m.ask_id,
            status: m.status,
            created_at: m.created_at,
            updated_at: m.updated_at,
            ask_title: m.ask_title || 'Opportunity',
            title: m.ask_title || 'Opportunity', // Alias for frontend
            ask_text: m.ask_text,
            category: m.category,
            requester_name: m.requester_name || 'Someone',
            other_name: m.requester_name || 'Someone', // Alias for frontend
            requester_id: m.requester_id
        }));
        
        console.log('[Opportunities] Found', opportunities.length, 'archived for user', user.id);
        
        res.json({
            success: true,
            opportunities: opportunities,
            count: opportunities.length
        });
        
    } catch (error) {
        console.error('[Opportunities] Archive error:', error);
        res.status(500).json({ success: false, error: error.message });
    }
});
```

---

## PART 4: Update User Profile/Login to Return referred_opportunity_id

### Location: `routes/users.js` (profile endpoint)

Update any user profile/login endpoints to return referral data:

```javascript
// GET /api/users/:identifier/profile
router.get('/users/:identifier/profile', async (req, res) => {
    try {
        const { identifier } = req.params;
        
        // Resolve user (phone or ID)
        let user;
        if (isNaN(identifier)) {
            const result = await client.query(
                'SELECT * FROM users WHERE phone = $1',
                [identifier]
            );
            user = result.rows[0];
        } else {
            const result = await client.query(
                'SELECT * FROM users WHERE id = $1',
                [parseInt(identifier)]
            );
            user = result.rows[0];
        }
        
        if (!user) {
            return res.status(404).json({ 
                success: false, 
                error: 'User not found' 
            });
        }
        
        res.json({
            success: true,
            user: {
                id: user.id,
                name: user.name,
                phone: user.phone,
                age: user.age,
                referred_by: user.referred_by,
                referred_opportunity_id: user.referred_opportunity_id
            }
        });
        
    } catch (error) {
        console.error('[Users] Profile error:', error);
        res.status(500).json({ success: false, error: error.message });
    }
});
```

---

## Database Schema Updates

Run these SQL migrations to ensure your database supports referrals:

```sql
-- Add referral columns to users table
ALTER TABLE users 
ADD COLUMN IF NOT EXISTS referred_by INTEGER REFERENCES users(id),
ADD COLUMN IF NOT EXISTS referred_opportunity_id INTEGER REFERENCES asks(id);

-- Update matches status constraint
ALTER TABLE matches 
DROP CONSTRAINT IF EXISTS matches_status_check;

ALTER TABLE matches 
ADD CONSTRAINT matches_status_check 
CHECK (status IN ('pending', 'accepted', 'declined', 'referred', 'expired'));

-- Add indexes for performance
CREATE INDEX IF NOT EXISTS idx_users_referred_by ON users(referred_by);
CREATE INDEX IF NOT EXISTS idx_users_referred_opportunity ON users(referred_opportunity_id);
CREATE INDEX IF NOT EXISTS idx_matches_status ON matches(status);
CREATE INDEX IF NOT EXISTS idx_matches_matched_user_status ON matches(matched_user_id, status);
```

---

## Testing Checklist

After implementing these changes, test:

1. ✅ Signup with referral data (`referred_by` and `referred_opportunity_id`)
2. ✅ Signup without referral data (should work normally)
3. ✅ Update match status to 'referred'
4. ✅ Get archived opportunities (should return declined + referred)
5. ✅ User profile returns `referred_opportunity_id`
6. ✅ Database constraints allow 'referred' status

---

## Notes

- The frontend expects the archive endpoint at: `/api/opportunities/:userId/archive`
- The frontend sends referral data in signup at: `/api/signup`
- The frontend updates match status at: `/api/matches/:id/respond`
- All endpoints should return `{ success: true/false, ... }` format

