---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for risk detection, task tracking, and ticket management
triggers:
  - "set up enterprise user management system"
  - "implement AI-powered user analytics"
  - "create task management with kanban board"
  - "build ticket classification system"
  - "integrate AI risk detection"
  - "develop user dashboard with time tracking"
  - "configure ML service for burnout detection"
  - "implement JWT authentication for user management"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

The Enterprise User Management System is a full-stack application that combines traditional user/task management with AI-powered analytics. It provides:

- **User Management**: Role-based access control, authentication, and user CRUD operations
- **Task Tracking**: Kanban boards, time tracking, and progress monitoring
- **Support Tickets**: AI-powered classification and routing
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, and project insights
- **Admin Dashboard**: Centralized monitoring and audit logs

## Architecture

The system consists of three main components:

1. **Frontend** (React.js) - User interface on port 3000
2. **Backend** (Node.js) - REST API on port 5000
3. **ML Service** (FastAPI) - AI analytics on port 8000

## Installation

### Prerequisites

```bash
# Node.js 14+ and Python 3.8+ required
node --version
python --version
```

### Full Setup

```bash
# Clone repository
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics

# Setup backend
cd backend
npm install
cp .env.example .env  # Configure environment variables

# Setup ML service
cd ../ml-service
pip install -r requirements.txt

# Setup frontend
cd ../frontend
npm install
```

### Environment Configuration

**Backend (.env)**:
```bash
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
JWT_SECRET=your_jwt_secret_key_here
JWT_EXPIRE=7d
NODE_ENV=development
ML_SERVICE_URL=http://localhost:8000
```

**ML Service (.env)**:
```bash
MODEL_PATH=./models
LOG_LEVEL=INFO
BACKEND_URL=http://localhost:5000
```

## Running the System

### Start All Services

```bash
# Terminal 1 - Backend
cd backend
npm start

# Terminal 2 - ML Service
cd ml-service
uvicorn main:app --reload --port 8000

# Terminal 3 - Frontend
cd frontend
npm start
```

### Production Mode

```bash
# Backend
cd backend
npm run build
npm run start:prod

# Frontend
cd frontend
npm run build
serve -s build

# ML Service
cd ml-service
uvicorn main:app --host 0.0.0.0 --port 8000
```

## Key Backend API Endpoints

### Authentication

```javascript
// Register new user
POST /api/auth/register
{
  "username": "johndoe",
  "email": "john@example.com",
  "password": "SecurePass123!",
  "role": "user"
}

// Login
POST /api/auth/login
{
  "email": "john@example.com",
  "password": "SecurePass123!"
}

// Response includes JWT token
{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "user": { "id": "...", "email": "...", "role": "user" }
}
```

### User Management (Admin)

```javascript
// Get all users
GET /api/users
Headers: { "Authorization": "Bearer <token>" }

// Update user
PUT /api/users/:id
{
  "role": "admin",
  "status": "active"
}

// Delete user
DELETE /api/users/:id
```

### Task Management

```javascript
// Create task
POST /api/tasks
{
  "title": "Implement authentication",
  "description": "Add JWT-based auth",
  "assignedTo": "user_id",
  "priority": "high",
  "status": "todo",
  "dueDate": "2026-05-01"
}

// Update task status
PATCH /api/tasks/:id/status
{
  "status": "in-progress"
}

// Get user tasks
GET /api/tasks/user/:userId
```

### Support Tickets

```javascript
// Create ticket
POST /api/tickets
{
  "title": "Cannot access dashboard",
  "description": "Getting 403 error",
  "priority": "high",
  "category": "technical"
}

// Get tickets
GET /api/tickets?status=open&priority=high
```

## ML Service API

### Risk Prediction

```python
# Python client example
import requests

def predict_user_risk(user_data):
    response = requests.post(
        "http://localhost:8000/api/ml/risk-prediction",
        json={
            "userId": user_data["id"],
            "loginFrequency": user_data["login_count"],
            "taskCompletionRate": user_data["completion_rate"],
            "failedLoginAttempts": user_data["failed_logins"],
            "unusualActivityScore": user_data["activity_score"]
        }
    )
    return response.json()

# Response
{
  "riskScore": 0.73,
  "riskLevel": "high",
  "factors": ["Multiple failed logins", "Low task completion"],
  "recommendations": ["Review account activity", "Reset password"]
}
```

### Anomaly Detection

```javascript
// JavaScript frontend example
const detectAnomaly = async (userData) => {
  const response = await fetch('http://localhost:8000/api/ml/anomaly-detection', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      userId: userData.id,
      activityPattern: userData.recentActivity,
      timestamp: new Date().toISOString()
    })
  });
  
  const result = await response.json();
  // { isAnomaly: true, confidence: 0.89, reason: "Unusual login time" }
  return result;
};
```

### Burnout Detection

```python
POST /api/ml/burnout-detection
{
  "userId": "user123",
  "workHours": [8, 9, 10, 12, 11, 10, 8],
  "tasksCompleted": [5, 6, 4, 3, 2, 2, 1],
  "overtimeHours": 15,
  "ticketsRaised": 8
}

# Response
{
  "burnoutRisk": "moderate",
  "score": 0.65,
  "indicators": ["Increasing overtime", "Declining productivity"],
  "suggestions": ["Redistribute workload", "Schedule time off"]
}
```

### Ticket Classification

```javascript
// Auto-classify and route ticket
const classifyTicket = async (ticketContent) => {
  const response = await fetch('http://localhost:8000/api/ml/classify-ticket', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      title: ticketContent.title,
      description: ticketContent.description,
      metadata: ticketContent.attachments
    })
  });
  
  return await response.json();
  // { category: "technical", priority: "high", assignTo: "team_backend" }
};
```

## Frontend Integration Patterns

### Authentication Hook

```javascript
// hooks/useAuth.js
import { useState, useEffect } from 'react';
import axios from 'axios';

export const useAuth = () => {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    const token = localStorage.getItem('token');
    if (token) {
      axios.defaults.headers.common['Authorization'] = `Bearer ${token}`;
      fetchUser();
    } else {
      setLoading(false);
    }
  }, []);

  const fetchUser = async () => {
    try {
      const response = await axios.get('http://localhost:5000/api/auth/me');
      setUser(response.data);
    } catch (error) {
      localStorage.removeItem('token');
    } finally {
      setLoading(false);
    }
  };

  const login = async (email, password) => {
    const response = await axios.post('http://localhost:5000/api/auth/login', {
      email,
      password
    });
    localStorage.setItem('token', response.data.token);
    setUser(response.data.user);
    return response.data;
  };

  const logout = () => {
    localStorage.removeItem('token');
    delete axios.defaults.headers.common['Authorization'];
    setUser(null);
  };

  return { user, loading, login, logout };
};
```

### Kanban Board Component

```javascript
// components/KanbanBoard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const KanbanBoard = ({ userId }) => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });

  useEffect(() => {
    fetchTasks();
  }, [userId]);

  const fetchTasks = async () => {
    const response = await axios.get(`http://localhost:5000/api/tasks/user/${userId}`);
    const grouped = {
      todo: response.data.filter(t => t.status === 'todo'),
      inProgress: response.data.filter(t => t.status === 'in-progress'),
      done: response.data.filter(t => t.status === 'done')
    };
    setTasks(grouped);
  };

  const moveTask = async (taskId, newStatus) => {
    await axios.patch(`http://localhost:5000/api/tasks/${taskId}/status`, {
      status: newStatus
    });
    fetchTasks();
  };

  return (
    <div className="kanban-board">
      {['todo', 'inProgress', 'done'].map(status => (
        <div key={status} className="kanban-column">
          <h3>{status.toUpperCase()}</h3>
          {tasks[status].map(task => (
            <TaskCard key={task.id} task={task} onMove={moveTask} />
          ))}
        </div>
      ))}
    </div>
  );
};
```

### AI Analytics Dashboard

```javascript
// components/AIAnalyticsDashboard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const AIAnalyticsDashboard = () => {
  const [analytics, setAnalytics] = useState(null);

  useEffect(() => {
    fetchAnalytics();
  }, []);

  const fetchAnalytics = async () => {
    const [riskData, burnoutData, anomalies] = await Promise.all([
      axios.get('http://localhost:8000/api/ml/risk-summary'),
      axios.get('http://localhost:8000/api/ml/burnout-summary'),
      axios.get('http://localhost:8000/api/ml/anomalies')
    ]);

    setAnalytics({
      risks: riskData.data,
      burnout: burnoutData.data,
      anomalies: anomalies.data
    });
  };

  if (!analytics) return <div>Loading analytics...</div>;

  return (
    <div className="ai-dashboard">
      <div className="risk-alerts">
        <h2>High Risk Users</h2>
        {analytics.risks.highRisk.map(user => (
          <RiskCard key={user.id} user={user} />
        ))}
      </div>
      
      <div className="burnout-warnings">
        <h2>Burnout Risk</h2>
        {analytics.burnout.atRisk.map(user => (
          <BurnoutCard key={user.id} user={user} />
        ))}
      </div>
      
      <div className="anomaly-alerts">
        <h2>Recent Anomalies</h2>
        {analytics.anomalies.recent.map(anomaly => (
          <AnomalyCard key={anomaly.id} anomaly={anomaly} />
        ))}
      </div>
    </div>
  );
};
```

## Database Schema

### User Model

```javascript
// models/User.js
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');

const userSchema = new mongoose.Schema({
  username: { type: String, required: true, unique: true },
  email: { type: String, required: true, unique: true },
  password: { type: String, required: true },
  role: { type: String, enum: ['user', 'admin'], default: 'user' },
  status: { type: String, enum: ['active', 'inactive'], default: 'active' },
  profile: {
    firstName: String,
    lastName: String,
    department: String,
    position: String
  },
  metadata: {
    lastLogin: Date,
    loginCount: { type: Number, default: 0 },
    failedLoginAttempts: { type: Number, default: 0 }
  }
}, { timestamps: true });

userSchema.pre('save', async function(next) {
  if (!this.isModified('password')) return next();
  this.password = await bcrypt.hash(this.password, 10);
  next();
});

module.exports = mongoose.model('User', userSchema);
```

### Task Model

```javascript
// models/Task.js
const taskSchema = new mongoose.Schema({
  title: { type: String, required: true },
  description: String,
  status: { 
    type: String, 
    enum: ['todo', 'in-progress', 'done'], 
    default: 'todo' 
  },
  priority: { 
    type: String, 
    enum: ['low', 'medium', 'high'], 
    default: 'medium' 
  },
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  createdBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  dueDate: Date,
  timeTracking: {
    estimatedHours: Number,
    actualHours: { type: Number, default: 0 },
    sessions: [{
      startTime: Date,
      endTime: Date,
      duration: Number
    }]
  }
}, { timestamps: true });
```

## Common Patterns

### Protected Routes

```javascript
// middleware/auth.js
const jwt = require('jsonwebtoken');

const authMiddleware = (req, res, next) => {
  const token = req.header('Authorization')?.replace('Bearer ', '');
  
  if (!token) {
    return res.status(401).json({ error: 'Access denied' });
  }

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch (error) {
    res.status(401).json({ error: 'Invalid token' });
  }
};

const adminOnly = (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ error: 'Admin access required' });
  }
  next();
};

module.exports = { authMiddleware, adminOnly };
```

### ML Model Integration

```python
# ml-service/services/risk_predictor.py
from river import linear_model, preprocessing, compose
import joblib
import os

class RiskPredictor:
    def __init__(self):
        self.model = compose.Pipeline(
            preprocessing.StandardScaler(),
            linear_model.LogisticRegression()
        )
        self.load_model()
    
    def load_model(self):
        model_path = os.getenv('MODEL_PATH', './models/risk_model.pkl')
        if os.path.exists(model_path):
            self.model = joblib.load(model_path)
    
    def predict(self, features):
        """
        features = {
            'login_frequency': float,
            'failed_logins': int,
            'task_completion_rate': float,
            'unusual_activity': float
        }
        """
        risk_score = self.model.predict_proba_one(features)
        return {
            'risk_score': risk_score.get(1, 0),
            'risk_level': self._get_risk_level(risk_score.get(1, 0))
        }
    
    def _get_risk_level(self, score):
        if score >= 0.7: return 'high'
        if score >= 0.4: return 'moderate'
        return 'low'
    
    def train_online(self, features, label):
        """Incremental learning"""
        self.model.learn_one(features, label)
```

## Troubleshooting

### Connection Issues

```bash
# Backend can't connect to MongoDB
# Check MongoDB is running
sudo systemctl status mongod

# Or use Docker
docker run -d -p 27017:27017 --name mongodb mongo:latest

# Verify connection
mongo --eval "db.adminCommand('ping')"
```

### JWT Token Expiration

```javascript
// Implement token refresh
const refreshToken = async () => {
  const refreshToken = localStorage.getItem('refreshToken');
  const response = await axios.post('/api/auth/refresh', { refreshToken });
  localStorage.setItem('token', response.data.token);
  return response.data.token;
};

// Axios interceptor for auto-refresh
axios.interceptors.response.use(
  response => response,
  async error => {
    if (error.response?.status === 401) {
      const newToken = await refreshToken();
      error.config.headers['Authorization'] = `Bearer ${newToken}`;
      return axios.request(error.config);
    }
    return Promise.reject(error);
  }
);
```

### ML Service Memory Issues

```bash
# For large datasets, use batch processing
# ml-service/config.py
BATCH_SIZE = 100
MAX_MEMORY_MB = 512

# Process in chunks
def process_large_dataset(data):
    for i in range(0, len(data), BATCH_SIZE):
        batch = data[i:i + BATCH_SIZE]
        yield process_batch(batch)
```

### CORS Issues

```javascript
// backend/app.js
const cors = require('cors');

app.use(cors({
  origin: process.env.FRONTEND_URL || 'http://localhost:3000',
  credentials: true
}));
```

## Performance Optimization

### Database Indexing

```javascript
// Add indexes for frequent queries
userSchema.index({ email: 1 });
userSchema.index({ role: 1, status: 1 });
taskSchema.index({ assignedTo: 1, status: 1 });
taskSchema.index({ dueDate: 1 });
```

### API Response Caching

```javascript
const NodeCache = require('node-cache');
const cache = new NodeCache({ stdTTL: 600 }); // 10 minutes

app.get('/api/analytics/summary', async (req, res) => {
  const cacheKey = 'analytics_summary';
  const cached = cache.get(cacheKey);
  
  if (cached) return res.json(cached);
  
  const data = await generateAnalytics();
  cache.set(cacheKey, data);
  res.json(data);
});
```

This skill provides comprehensive coverage of the Enterprise User Management System with AI Analytics, enabling AI coding agents to effectively assist developers in implementing, configuring, and troubleshooting the system.
