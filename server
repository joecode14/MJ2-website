// server.js - Complete Backend for MWIRUTI JNR
const express = require('express');
const cors = require('cors');
const path = require('path');
const { Pool } = require('pg');
const multer = require('multer');
const fs = require('fs-extra');
const bcrypt = require('bcryptjs');
const jwt = require('jsonwebtoken');
require('dotenv').config();

const app = express();

// Middleware
app.use(cors());
app.use(express.json());
app.use(express.static(path.join(__dirname, 'public')));

// PostgreSQL Connection for Render
const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
  ssl: process.env.NODE_ENV === 'production' ? { rejectUnauthorized: false } : false
});

// Create uploads directory
const uploadsDir = path.join(__dirname, 'public', 'uploads');
fs.ensureDirSync(uploadsDir);

// File upload configuration
const storage = multer.diskStorage({
  destination: (req, file, cb) => {
    cb(null, uploadsDir);
  },
  filename: (req, file, cb) => {
    const uniqueSuffix = Date.now() + '-' + Math.round(Math.random() * 1E9);
    cb(null, uniqueSuffix + path.extname(file.originalname));
  }
});

const upload = multer({
  storage: storage,
  limits: { fileSize: 5 * 1024 * 1024 }, // 5MB limit
  fileFilter: (req, file, cb) => {
    const allowedTypes = /jpeg|jpg|png|webp/;
    const extname = allowedTypes.test(path.extname(file.originalname).toLowerCase());
    const mimetype = allowedTypes.test(file.mimetype);
    
    if (mimetype && extname) {
      return cb(null, true);
    } else {
      cb(new Error('Only image files are allowed'));
    }
  }
});

// Initialize Database Tables
async function initializeDatabase() {
  try {
    await pool.query(`
      CREATE TABLE IF NOT EXISTS motorcycles (
        id SERIAL PRIMARY KEY,
        name VARCHAR(255) NOT NULL,
        price VARCHAR(100) NOT NULL,
        description TEXT,
        year VARCHAR(10),
        mileage VARCHAR(50),
        location VARCHAR(100),
        featured BOOLEAN DEFAULT true,
        created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
        updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
        deleted_at TIMESTAMP
      );

      CREATE TABLE IF NOT EXISTS motorcycle_images (
        id SERIAL PRIMARY KEY,
        motorcycle_id INTEGER REFERENCES motorcycles(id) ON DELETE CASCADE,
        image_url VARCHAR(500) NOT NULL,
        image_name VARCHAR(255),
        size INTEGER,
        uploaded_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
        is_primary BOOLEAN DEFAULT false
      );

      CREATE TABLE IF NOT EXISTS testimonials (
        id SERIAL PRIMARY KEY,
        name VARCHAR(100) NOT NULL,
        location VARCHAR(100),
        text TEXT NOT NULL,
        color VARCHAR(50) DEFAULT 'blue',
        created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
        deleted_at TIMESTAMP
      );

      CREATE TABLE IF NOT EXISTS inquiries (
        id SERIAL PRIMARY KEY,
        name VARCHAR(100) NOT NULL,
        phone VARCHAR(20) NOT NULL,
        model VARCHAR(100),
        year VARCHAR(10),
        details TEXT,
        photos_count INTEGER DEFAULT 0,
        created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
      );

      CREATE TABLE IF NOT EXISTS admin_users (
        id SERIAL PRIMARY KEY,
        username VARCHAR(50) UNIQUE NOT NULL,
        password_hash VARCHAR(255) NOT NULL,
        created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
      );

      -- Insert default admin if not exists
      INSERT INTO admin_users (username, password_hash) 
      SELECT 'admin', '$2a$10$92IXUNpkjO0rOQ5byMi.Ye4oKoEa3Ro9llC/.og/at2.uheWG/igi' -- password: 'mwirutijnr2025'
      WHERE NOT EXISTS (SELECT 1 FROM admin_users WHERE username = 'admin');
    `);
    console.log('Database tables initialized successfully');
  } catch (error) {
    console.error('Error initializing database:', error);
  }
}

// API Routes

// Test API
app.get('/api/health', (req, res) => {
  res.json({ 
    status: 'healthy',
    message: 'MWIRUTI JNR API is running',
    timestamp: new Date().toISOString()
  });
});

// Get all featured motorcycles
app.get('/api/motorcycles', async (req, res) => {
  try {
    const result = await pool.query(`
      SELECT m.*, 
             COALESCE(
               json_agg(
                 json_build_object(
                   'id', mi.id,
                   'image_url', mi.image_url,
                   'image_name', mi.image_name,
                   'size', mi.size,
                   'is_primary', mi.is_primary
                 ) ORDER BY mi.is_primary DESC, mi.uploaded_at ASC
               ) FILTER (WHERE mi.id IS NOT NULL), '[]'
             ) as images
      FROM motorcycles m
      LEFT JOIN motorcycle_images mi ON m.id = mi.motorcycle_id
      WHERE m.deleted_at IS NULL AND m.featured = true
      GROUP BY m.id
      ORDER BY m.created_at DESC
    `);
    res.json(result.rows);
  } catch (error) {
    console.error('Error fetching motorcycles:', error);
    res.status(500).json({ error: 'Internal server error' });
  }
});

// Create new motorcycle
app.post('/api/motorcycles', async (req, res) => {
  try {
    const { name, price, description, year, mileage, location, featured = true } = req.body;
    const result = await pool.query(
      `INSERT INTO motorcycles (name, price, description, year, mileage, location, featured) 
       VALUES ($1, $2, $3, $4, $5, $6, $7) RETURNING *`,
      [name, price, description, year, mileage, location, featured]
    );
    res.status(201).json(result.rows[0]);
  } catch (error) {
    console.error('Error creating motorcycle:', error);
    res.status(500).json({ error: 'Internal server error' });
  }
});

// Update motorcycle
app.put('/api/motorcycles/:id', async (req, res) => {
  try {
    const { id } = req.params;
    const { name, price, description, year, mileage, location, featured } = req.body;
    
    const result = await pool.query(
      `UPDATE motorcycles 
       SET name = $1, price = $2, description = $3, year = $4, 
           mileage = $5, location = $6, featured = $7, updated_at = CURRENT_TIMESTAMP
       WHERE id = $8 AND deleted_at IS NULL 
       RETURNING *`,
      [name, price, description, year, mileage, location, featured, id]
    );
    
    if (result.rows.length === 0) {
      return res.status(404).json({ error: 'Motorcycle not found' });
    }
    
    res.json(result.rows[0]);
  } catch (error) {
    console.error('Error updating motorcycle:', error);
    res.status(500).json({ error: 'Internal server error' });
  }
});

// Delete motorcycle (soft delete)
app.delete('/api/motorcycles/:id', async (req, res) => {
  try {
    const { id } = req.params;
    
    await pool.query(
      `UPDATE motorcycles SET deleted_at = CURRENT_TIMESTAMP WHERE id = $1`,
      [id]
    );
    
    res.json({ message: 'Motorcycle deleted successfully' });
  } catch (error) {
    console.error('Error deleting motorcycle:', error);
    res.status(500).json({ error: 'Internal server error' });
  }
});

// Upload motorcycle image
app.post('/api/motorcycles/:id/images', upload.array('images', 5), async (req, res) => {
  try {
    const { id } = req.params;
    const files = req.files;
    
    if (!files || files.length === 0) {
      return res.status(400).json({ error: 'No images uploaded' });
    }

    // Check if motorcycle exists
    const bikeCheck = await pool.query(
      'SELECT id FROM motorcycles WHERE id = $1 AND deleted_at IS NULL',
      [id]
    );
    
    if (bikeCheck.rows.length === 0) {
      // Clean up uploaded files
      files.forEach(file => fs.unlinkSync(file.path));
      return res.status(404).json({ error: 'Motorcycle not found' });
    }

    const baseUrl = `${req.protocol}://${req.get('host')}`;
    const imageInserts = [];
    
    for (const file of files) {
      const imageUrl = `${baseUrl}/uploads/${file.filename}`;
      
      const result = await pool.query(
        `INSERT INTO motorcycle_images (motorcycle_id, image_url, image_name, size) 
         VALUES ($1, $2, $3, $4) RETURNING *`,
        [id, imageUrl, file.originalname, file.size]
      );
      imageInserts.push(result.rows[0]);
    }
    
    res.status(201).json(imageInserts);
  } catch (error) {
    console.error('Error uploading images:', error);
    
    // Clean up uploaded files on error
    if (req.files) {
      req.files.forEach(file => {
        try {
          fs.unlinkSync(file.path);
        } catch (err) {
          console.error('Error cleaning up file:', err);
        }
      });
    }
    
    res.status(500).json({ error: 'Internal server error' });
  }
});

// Get testimonials
app.get('/api/testimonials', async (req, res) => {
  try {
    const result = await pool.query(`
      SELECT * FROM testimonials 
      WHERE deleted_at IS NULL 
      ORDER BY created_at DESC
    `);
    res.json(result.rows);
  } catch (error) {
    console.error('Error fetching testimonials:', error);
    res.status(500).json({ error: 'Internal server error' });
  }
});

// Create testimonial
app.post('/api/testimonials', async (req, res) => {
  try {
    const { name, location, text, color = 'blue' } = req.body;
    const result = await pool.query(
      `INSERT INTO testimonials (name, location, text, color) 
       VALUES ($1, $2, $3, $4) RETURNING *`,
      [name, location, text, color]
    );
    res.status(201).json(result.rows[0]);
  } catch (error) {
    console.error('Error creating testimonial:', error);
    res.status(500).json({ error: 'Internal server error' });
  }
});

// Update testimonial
app.put('/api/testimonials/:id', async (req, res) => {
  try {
    const { id } = req.params;
    const { name, location, text, color } = req.body;
    
    const result = await pool.query(
      `UPDATE testimonials 
       SET name = $1, location = $2, text = $3, color = $4
       WHERE id = $5 AND deleted_at IS NULL 
       RETURNING *`,
      [name, location, text, color, id]
    );
    
    if (result.rows.length === 0) {
      return res.status(404).json({ error: 'Testimonial not found' });
    }
    
    res.json(result.rows[0]);
  } catch (error) {
    console.error('Error updating testimonial:', error);
    res.status(500).json({ error: 'Internal server error' });
  }
});

// Delete testimonial
app.delete('/api/testimonials/:id', async (req, res) => {
  try {
    const { id } = req.params;
    
    await pool.query(
      `UPDATE testimonials SET deleted_at = CURRENT_TIMESTAMP WHERE id = $1`,
      [id]
    );
    
    res.json({ message: 'Testimonial deleted successfully' });
  } catch (error) {
    console.error('Error deleting testimonial:', error);
    res.status(500).json({ error: 'Internal server error' });
  }
});

// Submit inquiry
app.post('/api/inquiries', upload.array('photos', 5), async (req, res) => {
  try {
    const { name, phone, model, year, details } = req.body;
    const files = req.files || [];
    
    // Clean up uploaded files for inquiries (we don't store them permanently)
    files.forEach(file => {
      try {
        fs.unlinkSync(file.path);
      } catch (err) {
        console.error('Error cleaning up inquiry file:', err);
      }
    });

    const result = await pool.query(
      `INSERT INTO inquiries (name, phone, model, year, details, photos_count) 
       VALUES ($1, $2, $3, $4, $5, $6) RETURNING *`,
      [name, phone, model, year, details, files.length]
    );
    
    res.status(201).json(result.rows[0]);
  } catch (error) {
    console.error('Error submitting inquiry:', error);
    res.status(500).json({ error: 'Internal server error' });
  }
});

// Admin authentication
app.post('/api/admin/login', async (req, res) => {
  try {
    const { username, password } = req.body;
    
    const result = await pool.query(
      'SELECT * FROM admin_users WHERE username = $1',
      [username]
    );
    
    if (result.rows.length === 0) {
      return res.status(401).json({ error: 'Invalid credentials' });
    }
    
    const admin = result.rows[0];
    const isValid = await bcrypt.compare(password, admin.password_hash);
    
    if (!isValid) {
      return res.status(401).json({ error: 'Invalid credentials' });
    }
    
    // Create JWT token
    const token = jwt.sign(
      { id: admin.id, username: admin.username },
      process.env.JWT_SECRET || 'mwiruti-secret-key-2025',
      { expiresIn: '8h' }
    );
    
    res.json({
      token,
      expiry: new Date(Date.now() + 8 * 60 * 60 * 1000).toISOString(),
      user: { username: admin.username }
    });
  } catch (error) {
    console.error('Error in admin login:', error);
    res.status(500).json({ error: 'Internal server error' });
  }
});

// Verify admin token
app.post('/api/admin/verify', async (req, res) => {
  try {
    const { token } = req.body;
    
    if (!token) {
      return res.json({ valid: false });
    }
    
    const decoded = jwt.verify(
      token, 
      process.env.JWT_SECRET || 'mwiruti-secret-key-2025'
    );
    
    res.json({ valid: true, user: { username: decoded.username } });
  } catch (error) {
    res.json({ valid: false });
  }
});

// Backup data
app.get('/api/backup', async (req, res) => {
  try {
    const [motorcycles, testimonials] = await Promise.all([
      pool.query(`
        SELECT m.*, 
               COALESCE(
                 json_agg(
                   json_build_object(
                     'id', mi.id,
                     'image_url', mi.image_url,
                     'image_name', mi.image_name,
                     'size', mi.size,
                     'is_primary', mi.is_primary
                   ) ORDER BY mi.is_primary DESC, mi.uploaded_at ASC
                 ) FILTER (WHERE mi.id IS NOT NULL), '[]'
               ) as images
        FROM motorcycles m
        LEFT JOIN motorcycle_images mi ON m.id = mi.motorcycle_id
        WHERE m.deleted_at IS NULL
        GROUP BY m.id
        ORDER BY m.created_at DESC
      `),
      pool.query('SELECT * FROM testimonials WHERE deleted_at IS NULL')
    ]);
    
    const backupData = {
      motorcycles: motorcycles.rows,
      testimonials: testimonials.rows,
      backup_date: new Date().toISOString(),
      version: '1.0'
    };
    
    res.json(backupData);
  } catch (error) {
    console.error('Error creating backup:', error);
    res.status(500).json({ error: 'Internal server error' });
  }
});

// Serve frontend for all other routes
app.get('*', (req, res) => {
  res.sendFile(path.join(__dirname, 'public', 'index.html'));
});

// Initialize database and start server
const PORT = process.env.PORT || 3000;

async function startServer() {
  await initializeDatabase();
  
  app.listen(PORT, () => {
    console.log(`
    =============================================
    MWIRUTI JNR SERVER STARTED SUCCESSFULLY!
    =============================================
    Server running on port: ${PORT}
    API Base URL: http://localhost:${PORT}/api
    Frontend URL: http://localhost:${PORT}
    Uploads Directory: ${uploadsDir}
    =============================================
    `);
  });
}

startServer();