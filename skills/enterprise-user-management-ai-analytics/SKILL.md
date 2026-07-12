---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for task tracking, ticket management, and burnout detection
triggers:
  - "set up enterprise user management system"
  - "implement AI analytics for user management"
  - "create admin dashboard with user analytics"
  - "build task management with AI insights"
  - "add burnout detection to user system"
  - "configure JWT authentication for user management"
  - "integrate ML service for ticket classification"
  - "deploy user management system with AI"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## What This Project Does

Enterprise User Management System with AI Analytics is a full-stack web application that combines traditional user/task management with AI-powered insights. It provides:

- **User Management**: JWT-based authentication, role-based access control (Admin/User)
- **Task Tracking**: Kanban board (To Do → In Progress → Done) with time tracking
- **Support Tickets**: AI-classified and auto-routed support system
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, project delay prediction
- **Admin Dashboard**: Organization-wide analytics, audit logs, user monitoring

**Stack**: React frontend, Node.js/Express backend, FastAPI ML service, MongoDB database

## Installation

### Prerequisites

```bash
# Required
node >= 14.x
npm >= 6.x
python >= 3.8
mongodb >= 4.x
```

### Clone and Setup

```bash
# Clone repository
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics
```

### Backend Setup

```bash
cd backend
npm install

# Create .env file
cat > .env << EOF
PORT=5000
MONGODB_URI=mongodb://localhost:27017/user-management
JWT_SECRET=${JWT_SECRET}
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
EOF

# Start backend
npm start
# Runs at http://localhost:5000
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt

# Create .env file
cat > .env << EOF
MONGODB_URI=mongodb://localhost:27017/user-management
MODEL_PATH=./models
LOG_LEVEL=INFO
EOF

# Start ML service
uvicorn main:app --reload --port 8000
# Runs at http://localhost:8000
```

### Frontend Setup

```bash
cd frontend
npm install

# Create .env file
cat > .env << EOF
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_URL=http://localhost:8000
EOF

# Start frontend
npm start
# Runs at http://localhost:3000
```

## Key Architecture Components

### Backend API Structure (Node.js/Express)

```javascript
// backend/server.js
const express = require('express');
const mongoose = require('mongoose');
const cors = require('cors');
require('dotenv').config();

const app = express();

// Middleware
app.use(cors());
app.use(express.json());

// Database connection
mongoose.connect(process.env.MONGODB_URI, {
  useNewUrlParser: true,
  useUnifiedTopology: true
});

// Routes
app.use('/api/auth', require('./routes/auth'));
app.use('/api/users', require('./routes/users'));
app.use('/api/tasks', require('./routes/tasks'));
app.use('/api/tickets', require('./routes/tickets'));
app.use('/api/analytics', require('./routes/analytics'));

const PORT = process.env.PORT || 5000;
app.listen(PORT, () => console.log(`Server running on port ${PORT}`));
```

### Authentication Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

const auth = async (req, res, next) => {
  try {
    const token = req.header('Authorization')?.replace('Bearer ', '');
    
    if (!token) {
      return res.status(401).json({ error: 'No authentication token' });
    }

    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    const user = await User.findById(decoded.id);

    if (!user) {
      return res.status(401).json({ error: 'User not found' });
    }

    req.user = user;
    req.token = token;
    next();
  } catch (error) {
    res.status(401).json({ error: 'Authentication failed' });
  }
};

const adminAuth = async (req, res, next) => {
  await auth(req, res, () => {
    if (req.user.role !== 'admin') {
      return res.status(403).json({ error: 'Admin access required' });
    }
    next();
  });
};

module.exports = { auth, adminAuth };
```

### User Model

```javascript
// backend/models/User.js
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');

const userSchema = new mongoose.Schema({
  name: { type: String, required: true },
  email: { type: String, required: true, unique: true },
  password: { type: String, required: true },
  role: { type: String, enum: ['user', 'admin'], default: 'user' },
  department: String,
  position: String,
  status: { type: String, enum: ['active', 'inactive'], default: 'active' },
  lastLogin: Date,
  createdAt: { type: Date, default: Date.now }
});

// Hash password before saving
userSchema.pre('save', async function(next) {
  if (this.isModified('password')) {
    this.password = await bcrypt.hash(this.password, 10);
  }
  next();
});

// Compare password method
userSchema.methods.comparePassword = async function(candidatePassword) {
  return await bcrypt.compare(candidatePassword, this.password);
};

module.exports = mongoose.model('User', userSchema);
```

### Task Model with AI Features

```javascript
// backend/models/Task.js
const mongoose = require('mongoose');

const taskSchema = new mongoose.Schema({
  title: { type: String, required: true },
  description: String,
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  createdBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  status: { 
    type: String, 
    enum: ['todo', 'inprogress', 'done'], 
    default: 'todo' 
  },
  priority: { 
    type: String, 
    enum: ['low', 'medium', 'high', 'critical'], 
    default: 'medium' 
  },
  dueDate: Date,
  timeTracked: { type: Number, default: 0 }, // in minutes
  aiRiskScore: { type: Number, min: 0, max: 100 },
  aiPredictedDelay: Boolean,
  tags: [String],
  createdAt: { type: Date, default: Date.now },
  updatedAt: { type: Date, default: Date.now }
});

module.exports = mongoose.model('Task', taskSchema);
```

### Ticket Model with AI Classification

```javascript
// backend/models/Ticket.js
const mongoose = require('mongoose');

const ticketSchema = new mongoose.Schema({
  title: { type: String, required: true },
  description: { type: String, required: true },
  category: { 
    type: String, 
    enum: ['technical', 'billing', 'general', 'urgent', 'feedback'],
    default: 'general'
  },
  priority: { 
    type: String, 
    enum: ['low', 'medium', 'high', 'critical'] 
  },
  status: { 
    type: String, 
    enum: ['open', 'in-progress', 'resolved', 'closed'], 
    default: 'open' 
  },
  createdBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  aiClassified: { type: Boolean, default: false },
  aiConfidence: Number,
  createdAt: { type: Date, default: Date.now },
  resolvedAt: Date
});

module.exports = mongoose.model('Ticket', ticketSchema);
```

## ML Service API (FastAPI)

### Main ML Service

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import List, Optional
import numpy as np
from datetime import datetime
import pickle
import os

app = FastAPI(title="User Management AI Service")

# Load or initialize models
MODEL_PATH = os.getenv('MODEL_PATH', './models')

class TicketClassificationRequest(BaseModel):
    title: str
    description: str

class TicketClassificationResponse(BaseModel):
    category: str
    priority: str
    confidence: float

class BurnoutAnalysisRequest(BaseModel):
    user_id: str
    tasks_count: int
    avg_task_duration: float
    overtime_hours: float
    missed_deadlines: int

class BurnoutAnalysisResponse(BaseModel):
    burnout_risk: str  # low, medium, high
    risk_score: float
    recommendations: List[str]

@app.get("/")
def read_root():
    return {"service": "User Management AI Analytics", "status": "running"}

@app.post("/classify-ticket", response_model=TicketClassificationResponse)
async def classify_ticket(request: TicketClassificationRequest):
    """AI-based ticket classification"""
    try:
        # Simple keyword-based classification (replace with ML model)
        text = f"{request.title} {request.description}".lower()
        
        # Category classification
        if any(word in text for word in ['bug', 'error', 'crash', 'not working']):
            category = 'technical'
            priority = 'high'
        elif any(word in text for word in ['urgent', 'asap', 'critical']):
            category = 'urgent'
            priority = 'critical'
        elif any(word in text for word in ['payment', 'invoice', 'charge']):
            category = 'billing'
            priority = 'medium'
        elif any(word in text for word in ['suggestion', 'feedback', 'improve']):
            category = 'feedback'
            priority = 'low'
        else:
            category = 'general'
            priority = 'medium'
        
        confidence = 0.85  # Mock confidence score
        
        return TicketClassificationResponse(
            category=category,
            priority=priority,
            confidence=confidence
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/analyze-burnout", response_model=BurnoutAnalysisResponse)
async def analyze_burnout(request: BurnoutAnalysisRequest):
    """Analyze user burnout risk based on workload metrics"""
    try:
        # Calculate burnout score (0-100)
        score = 0
        
        # Task overload factor
        if request.tasks_count > 20:
            score += 25
        elif request.tasks_count > 10:
            score += 15
        
        # Overtime factor
        if request.overtime_hours > 40:
            score += 30
        elif request.overtime_hours > 20:
            score += 20
        
        # Missed deadlines factor
        score += min(request.missed_deadlines * 10, 25)
        
        # Average task duration (longer = more complex/stressful)
        if request.avg_task_duration > 480:  # 8 hours
            score += 20
        
        # Determine risk level
        if score >= 70:
            risk_level = 'high'
            recommendations = [
                "Reduce workload immediately",
                "Redistribute tasks to team members",
                "Schedule mandatory time off",
                "Consider mental health support"
            ]
        elif score >= 40:
            risk_level = 'medium'
            recommendations = [
                "Monitor workload closely",
                "Avoid assigning urgent tasks",
                "Encourage regular breaks",
                "Review task priorities"
            ]
        else:
            risk_level = 'low'
            recommendations = [
                "Maintain current workload balance",
                "Continue regular check-ins"
            ]
        
        return BurnoutAnalysisResponse(
            burnout_risk=risk_level,
            risk_score=float(score),
            recommendations=recommendations
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/predict-project-delay")
async def predict_project_delay(
    tasks_remaining: int,
    avg_completion_rate: float,
    days_until_deadline: int
):
    """Predict if project will be delayed"""
    try:
        estimated_days_needed = tasks_remaining / max(avg_completion_rate, 0.1)
        delay_risk = estimated_days_needed > days_until_deadline
        
        return {
            "will_delay": delay_risk,
            "estimated_days_needed": estimated_days_needed,
            "risk_percentage": min((estimated_days_needed / days_until_deadline) * 100, 100)
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/detect-anomaly")
async def detect_anomaly(
    login_time: str,
    login_location: str,
    failed_attempts: int
):
    """Detect anomalous user behavior"""
    try:
        anomaly_score = 0
        alerts = []
        
        # Parse login time
        hour = int(login_time.split(':')[0])
        if hour < 6 or hour > 22:
            anomaly_score += 30
            alerts.append("Unusual login time")
        
        # Failed login attempts
        if failed_attempts > 3:
            anomaly_score += 40
            alerts.append(f"Multiple failed login attempts: {failed_attempts}")
        
        # Location check (simplified - would use IP geolocation)
        if "unknown" in login_location.lower():
            anomaly_score += 30
            alerts.append("Login from unknown location")
        
        is_anomaly = anomaly_score >= 50
        
        return {
            "is_anomaly": is_anomaly,
            "anomaly_score": anomaly_score,
            "alerts": alerts,
            "recommended_action": "Block account" if anomaly_score >= 70 else "Monitor" if is_anomaly else "Allow"
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

## Frontend React Components

### User Dashboard Component

```javascript
// frontend/src/components/UserDashboard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const UserDashboard = () => {
  const [tasks, setTasks] = useState({ todo: [], inprogress: [], done: [] });
  const [stats, setStats] = useState({});
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchUserData();
  }, []);

  const fetchUserData = async () => {
    try {
      const token = localStorage.getItem('token');
      const config = { headers: { Authorization: `Bearer ${token}` } };
      
      const tasksRes = await axios.get(
        `${process.env.REACT_APP_API_URL}/tasks/my-tasks`,
        config
      );
      
      // Group tasks by status
      const grouped = tasksRes.data.reduce((acc, task) => {
        acc[task.status] = acc[task.status] || [];
        acc[task.status].push(task);
        return acc;
      }, { todo: [], inprogress: [], done: [] });
      
      setTasks(grouped);
      
      // Calculate stats
      setStats({
        total: tasksRes.data.length,
        completed: grouped.done.length,
        inProgress: grouped.inprogress.length
      });
      
      setLoading(false);
    } catch (error) {
      console.error('Error fetching data:', error);
      setLoading(false);
    }
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    try {
      const token = localStorage.getItem('token');
      await axios.patch(
        `${process.env.REACT_APP_API_URL}/tasks/${taskId}`,
        { status: newStatus },
        { headers: { Authorization: `Bearer ${token}` } }
      );
      fetchUserData();
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  if (loading) return <div>Loading...</div>;

  return (
    <div className="user-dashboard">
      <h1>My Dashboard</h1>
      
      <div className="stats-cards">
        <div className="stat-card">
          <h3>Total Tasks</h3>
          <p>{stats.total}</p>
        </div>
        <div className="stat-card">
          <h3>In Progress</h3>
          <p>{stats.inProgress}</p>
        </div>
        <div className="stat-card">
          <h3>Completed</h3>
          <p>{stats.completed}</p>
        </div>
      </div>

      <div className="kanban-board">
        {['todo', 'inprogress', 'done'].map(status => (
          <div key={status} className="kanban-column">
            <h2>{status.toUpperCase()}</h2>
            {tasks[status].map(task => (
              <div key={task._id} className="task-card">
                <h4>{task.title}</h4>
                <p>{task.description}</p>
                <span className={`priority ${task.priority}`}>
                  {task.priority}
                </span>
                {task.aiRiskScore && (
                  <span className="ai-risk">
                    Risk: {task.aiRiskScore}%
                  </span>
                )}
                <div className="task-actions">
                  {status !== 'done' && (
                    <button onClick={() => updateTaskStatus(
                      task._id, 
                      status === 'todo' ? 'inprogress' : 'done'
                    )}>
                      Move →
                    </button>
                  )}
                </div>
              </div>
            ))}
          </div>
        ))}
      </div>
    </div>
  );
};

export default UserDashboard;
```

### Admin Analytics Component

```javascript
// frontend/src/components/AdminAnalytics.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const AdminAnalytics = () => {
  const [analytics, setAnalytics] = useState(null);
  const [burnoutUsers, setBurnoutUsers] = useState([]);

  useEffect(() => {
    fetchAnalytics();
  }, []);

  const fetchAnalytics = async () => {
    try {
      const token = localStorage.getItem('token');
      const config = { headers: { Authorization: `Bearer ${token}` } };
      
      const analyticsRes = await axios.get(
        `${process.env.REACT_APP_API_URL}/analytics/overview`,
        config
      );
      setAnalytics(analyticsRes.data);
      
      // Fetch burnout analysis for all users
      const burnoutRes = await axios.get(
        `${process.env.REACT_APP_API_URL}/analytics/burnout-risks`,
        config
      );
      setBurnoutUsers(burnoutRes.data.filter(u => u.burnout_risk !== 'low'));
      
    } catch (error) {
      console.error('Error fetching analytics:', error);
    }
  };

  if (!analytics) return <div>Loading analytics...</div>;

  return (
    <div className="admin-analytics">
      <h1>Organization Analytics</h1>
      
      <div className="analytics-grid">
        <div className="analytics-card">
          <h3>Total Users</h3>
          <p className="big-number">{analytics.totalUsers}</p>
        </div>
        <div className="analytics-card">
          <h3>Active Tasks</h3>
          <p className="big-number">{analytics.activeTasks}</p>
        </div>
        <div className="analytics-card">
          <h3>Open Tickets</h3>
          <p className="big-number">{analytics.openTickets}</p>
        </div>
        <div className="analytics-card">
          <h3>Completion Rate</h3>
          <p className="big-number">{analytics.completionRate}%</p>
        </div>
      </div>

      <div className="burnout-alerts">
        <h2>Burnout Risk Alerts</h2>
        {burnoutUsers.length === 0 ? (
          <p>No high burnout risks detected</p>
        ) : (
          <table>
            <thead>
              <tr>
                <th>User</th>
                <th>Risk Level</th>
                <th>Risk Score</th>
                <th>Recommendations</th>
              </tr>
            </thead>
            <tbody>
              {burnoutUsers.map(user => (
                <tr key={user.user_id}>
                  <td>{user.name}</td>
                  <td className={`risk-${user.burnout_risk}`}>
                    {user.burnout_risk}
                  </td>
                  <td>{user.risk_score.toFixed(1)}</td>
                  <td>
                    <ul>
                      {user.recommendations.map((rec, i) => (
                        <li key={i}>{rec}</li>
                      ))}
                    </ul>
                  </td>
                </tr>
              ))}
            </tbody>
          </table>
        )}
      </div>
    </div>
  );
};

export default AdminAnalytics;
```

## API Integration Patterns

### Calling ML Service from Backend

```javascript
// backend/services/mlService.js
const axios = require('axios');

const ML_SERVICE_URL = process.env.ML_SERVICE_URL || 'http://localhost:8000';

class MLService {
  async classifyTicket(title, description) {
    try {
      const response = await axios.post(`${ML_SERVICE_URL}/classify-ticket`, {
        title,
        description
      });
      return response.data;
    } catch (error) {
      console.error('ML classification failed:', error.message);
      return null;
    }
  }

  async analyzeBurnout(userId, metrics) {
    try {
      const response = await axios.post(`${ML_SERVICE_URL}/analyze-burnout`, {
        user_id: userId,
        tasks_count: metrics.tasksCount,
        avg_task_duration: metrics.avgDuration,
        overtime_hours: metrics.overtimeHours,
        missed_deadlines: metrics.missedDeadlines
      });
      return response.data;
    } catch (error) {
      console.error('Burnout analysis failed:', error.message);
      return null;
    }
  }

  async detectAnomaly(loginData) {
    try {
      const response = await axios.post(`${ML_SERVICE_URL}/detect-anomaly`, {
        login_time: loginData.time,
        login_location: loginData.location,
        failed_attempts: loginData.failedAttempts
      });
      return response.data;
    } catch (error) {
      console.error('Anomaly detection failed:', error.message);
      return null;
    }
  }
}

module.exports = new MLService();
```

### Using ML Service in Ticket Routes

```javascript
// backend/routes/tickets.js
const express = require('express');
const router = express.Router();
const Ticket = require('../models/Ticket');
const { auth } = require('../middleware/auth');
const mlService = require('../services/mlService');

// Create ticket with AI classification
router.post('/', auth, async (req, res) => {
  try {
    const { title, description } = req.body;
    
    // Get AI classification
    const aiClassification = await mlService.classifyTicket(title, description);
    
    const ticket = new Ticket({
      title,
      description,
      createdBy: req.user._id,
      category: aiClassification?.category || 'general',
      priority: aiClassification?.priority || 'medium',
      aiClassified: !!aiClassification,
      aiConfidence: aiClassification?.confidence
    });
    
    await ticket.save();
    res.status(201).json(ticket);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

// Get user's tickets
router.get('/my-tickets', auth, async (req, res) => {
  try {
    const tickets = await Ticket.find({ createdBy: req.user._id })
      .sort({ createdAt: -1 });
    res.json(tickets);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

module.exports = router;
```

## Configuration

### Environment Variables

**Backend (.env)**
```env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/user-management
JWT_SECRET=${JWT_SECRET}
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
CORS_ORIGIN=http://localhost:3000
```

**Frontend (.env)**
```env
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_URL=http://localhost:8000
```

**ML Service (.env)**
```env
MONGODB_URI=mongodb://localhost:27017/user-management
MODEL_PATH=./models
LOG_LEVEL=INFO
```

## Common Patterns

### Protected Route Pattern (Frontend)

```javascript
// frontend/src/components/ProtectedRoute.jsx
import React from 'react';
import { Navigate } from 'react-router-dom';

const ProtectedRoute = ({ children, requireAdmin = false }) => {
  const token = localStorage.getItem('token');
  const user = JSON.parse(localStorage.getItem('user') || '{}');
  
  if (!token) {
    return <Navigate to="/login" />;
  }
  
  if (requireAdmin && user.role !== 'admin') {
    return <Navigate to="/dashboard" />;
  }
  
  return children;
};

export default ProtectedRoute;
```

### Axios Interceptor for Auth

```javascript
// frontend/src/utils/axiosConfig.js
import axios from 'axios';

const instance = axios.create({
  baseURL: process.env.REACT_APP_API_URL
});

instance.interceptors.request.use(
  config => {
    const token = localStorage.getItem('token');
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
  },
  error => Promise.reject(error)
);

instance.interceptors.response.use(
  response => response,
  error => {
    if (error.response?.status === 401) {
      localStorage.removeItem('token');
      localStorage.removeItem('user');
      window.location.href = '/login';
    }
    return Promise.reject(error);
  }
);

export default instance;
```

## Troubleshooting

### MongoDB Connection Issues

```bash
# Check MongoDB is running
sudo systemctl status mongodb

# Start MongoDB
sudo systemctl start mongodb

# Verify connection
mongo --eval "db.adminCommand('ping')"
```

### JWT Token Errors

```javascript
// Verify token manually
const jwt = require('jsonwebtoken');
const token = 'your-token-here';
try {
  const decoded = jwt.verify(token, process.env.JWT_SECRET);
  console.log('Token valid:', decoded);
} catch (error) {
  console.error('Token invalid:', error.message);
}
```

### ML Service Not Responding

```bash
# Check ML service is running
curl http://localhost:8000/

# Check logs
cd ml-service
tail -f uvicorn.log

# Reinstall dependencies if needed
pip install --force-reinstall -r requirements.txt
```

### CORS Issues

```javascript
// backend/server.js - Enable CORS properly
const cors = require('cors');
app.use(cors({
  origin: process.env.CORS_ORIGIN || 'http://localhost:3000',
  credentials: true
}));
```

### Frontend Build Errors

```bash
# Clear cache and rebuild
rm -rf node_modules package-lock.json
npm install
npm start

# Check environment variables are loaded
echo $REACT_APP_API_URL
```

### Database Seeding for Development

```javascript
// backend/scripts/seed.js
const mongoose = require('mongoose');
const User = require('../models/User');
require('dotenv').config();

async function seedDatabase() {
  await mongoose.connect(process.env.MONGODB_URI);
  
  // Create admin user
  await User.create({
    name: 'Admin User',
    email: 'admin@example.com',
    password: 'admin123',
    role: 'admin'
  });
  
  // Create regular users
  await User.create({
    name: 'John Doe',
    email: 'john@example.com',
    password: 'user123',
    role: 'user',
    department: 'Engineering'
  });
  
  console.log('Database seeded');
  process.exit(0);
}

seedDatabase();
```

Run seeding:
```bash
cd backend
node scripts/seed.js
```
