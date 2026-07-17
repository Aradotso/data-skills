---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics for risk detection, burnout analysis, and ticket classification
triggers:
  - "set up enterprise user management system"
  - "integrate AI analytics for user management"
  - "build user task tracking with AI insights"
  - "implement role-based user management dashboard"
  - "add AI ticket classification system"
  - "create kanban board with time tracking"
  - "deploy user management with anomaly detection"
  - "configure AI-powered support ticket routing"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A full-stack application combining user/task management with AI-powered analytics including risk detection, anomaly detection, burnout analysis, and predictive insights. Built with React frontend, Node.js backend, FastAPI ML service, and MongoDB.

## What This Project Does

- **User Management**: Role-based access control (Admin/User), authentication via JWT
- **Task Management**: Kanban boards (To Do → In Progress → Done) with time tracking
- **Support Tickets**: Submit, track, and auto-classify support requests
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, project delay prediction
- **Dashboard**: Real-time insights for admins and personalized views for users

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
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
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
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
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
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
EOF

# Start frontend
npm start
# Runs at http://localhost:3000
```

## Architecture Overview

```
frontend (React)
    ↓
backend (Node.js/Express) ← JWT Auth
    ↓
MongoDB ← User/Task/Ticket Data
    ↓
ml-service (FastAPI) ← AI Models
```

## Backend API Reference

### Authentication

```javascript
// Register new user
POST /api/auth/register
{
  "name": "John Doe",
  "email": "john@example.com",
  "password": "securepass123",
  "role": "user" // or "admin"
}

// Login
POST /api/auth/login
{
  "email": "john@example.com",
  "password": "securepass123"
}
// Returns: { token: "jwt_token", user: {...} }
```

### User Management

```javascript
// Get all users (Admin only)
GET /api/users
Headers: { Authorization: "Bearer ${JWT_TOKEN}" }

// Get user by ID
GET /api/users/:id

// Update user
PUT /api/users/:id
{
  "name": "John Updated",
  "role": "admin",
  "department": "Engineering"
}

// Delete user (Admin only)
DELETE /api/users/:id
```

### Task Management

```javascript
// Create task
POST /api/tasks
{
  "title": "Implement login feature",
  "description": "Add JWT authentication",
  "assignedTo": "user_id",
  "priority": "high",
  "status": "todo",
  "dueDate": "2026-05-01"
}

// Update task status
PATCH /api/tasks/:id/status
{
  "status": "in-progress" // or "done"
}

// Get user tasks
GET /api/tasks/user/:userId

// Track time on task
POST /api/tasks/:id/time
{
  "duration": 3600 // seconds
}
```

### Support Tickets

```javascript
// Create ticket
POST /api/tickets
{
  "title": "Cannot access dashboard",
  "description": "Getting 403 error",
  "priority": "medium",
  "category": "technical"
}

// Get all tickets (Admin)
GET /api/tickets

// Update ticket
PATCH /api/tickets/:id
{
  "status": "in-progress",
  "assignedTo": "admin_id",
  "notes": "Investigating permissions"
}
```

## ML Service API Reference

### Risk Detection

```javascript
// Predict user risk score
POST /api/ml/risk-detection
{
  "userId": "user_id",
  "failedLogins": 3,
  "taskCompletionRate": 0.65,
  "avgResponseTime": 48,
  "ticketsRaised": 12
}
// Returns: { riskScore: 0.72, riskLevel: "high", factors: [...] }
```

### Anomaly Detection

```javascript
// Detect anomalous behavior
POST /api/ml/anomaly-detection
{
  "userId": "user_id",
  "loginTime": "2026-04-15T03:30:00Z",
  "location": "Unknown",
  "activityPattern": [0, 0, 5, 2, 1, 0, 0]
}
// Returns: { isAnomaly: true, anomalyScore: 0.85, reason: "Unusual login time" }
```

### Burnout Analysis

```javascript
// Analyze burnout risk
POST /api/ml/burnout-detection
{
  "userId": "user_id",
  "weeklyHours": 65,
  "overtimeHours": 25,
  "taskLoad": 18,
  "missedDeadlines": 4,
  "stressLevel": 8
}
// Returns: { burnoutScore: 0.78, burnoutRisk: "high", recommendations: [...] }
```

### Ticket Classification

```javascript
// Auto-classify support ticket
POST /api/ml/classify-ticket
{
  "title": "Cannot reset password",
  "description": "Password reset email not arriving",
  "text": "I tried to reset my password but didn't receive the email"
}
// Returns: { category: "technical", priority: "medium", suggestedAssignee: "support_team" }
```

### Project Insights

```javascript
// Predict project delay
POST /api/ml/project-insights
{
  "projectId": "proj_123",
  "tasksTotal": 50,
  "tasksCompleted": 20,
  "daysRemaining": 30,
  "teamSize": 5,
  "avgTaskTime": 8
}
// Returns: { delayProbability: 0.65, estimatedDelay: 10, recommendation: "Add 2 resources" }
```

## Frontend Integration Patterns

### Authentication Context

```javascript
// src/context/AuthContext.js
import React, { createContext, useState, useEffect } from 'react';
import axios from 'axios';

export const AuthContext = createContext();

export const AuthProvider = ({ children }) => {
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
      const res = await axios.get(`${process.env.REACT_APP_API_URL}/api/auth/me`);
      setUser(res.data);
    } catch (error) {
      localStorage.removeItem('token');
    } finally {
      setLoading(false);
    }
  };

  const login = async (email, password) => {
    const res = await axios.post(`${process.env.REACT_APP_API_URL}/api/auth/login`, {
      email,
      password
    });
    localStorage.setItem('token', res.data.token);
    axios.defaults.headers.common['Authorization'] = `Bearer ${res.data.token}`;
    setUser(res.data.user);
    return res.data;
  };

  const logout = () => {
    localStorage.removeItem('token');
    delete axios.defaults.headers.common['Authorization'];
    setUser(null);
  };

  return (
    <AuthContext.Provider value={{ user, login, logout, loading }}>
      {children}
    </AuthContext.Provider>
  );
};
```

### Task Dashboard Component

```javascript
// src/components/TaskDashboard.js
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const TaskDashboard = ({ userId }) => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });

  useEffect(() => {
    fetchTasks();
  }, [userId]);

  const fetchTasks = async () => {
    try {
      const res = await axios.get(
        `${process.env.REACT_APP_API_URL}/api/tasks/user/${userId}`
      );
      const grouped = {
        todo: res.data.filter(t => t.status === 'todo'),
        inProgress: res.data.filter(t => t.status === 'in-progress'),
        done: res.data.filter(t => t.status === 'done')
      };
      setTasks(grouped);
    } catch (error) {
      console.error('Error fetching tasks:', error);
    }
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    try {
      await axios.patch(
        `${process.env.REACT_APP_API_URL}/api/tasks/${taskId}/status`,
        { status: newStatus }
      );
      fetchTasks();
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  return (
    <div className="kanban-board">
      {['todo', 'inProgress', 'done'].map(column => (
        <div key={column} className="kanban-column">
          <h3>{column.toUpperCase()}</h3>
          {tasks[column].map(task => (
            <div key={task._id} className="task-card">
              <h4>{task.title}</h4>
              <p>{task.description}</p>
              <select
                value={task.status}
                onChange={(e) => updateTaskStatus(task._id, e.target.value)}
              >
                <option value="todo">To Do</option>
                <option value="in-progress">In Progress</option>
                <option value="done">Done</option>
              </select>
            </div>
          ))}
        </div>
      ))}
    </div>
  );
};

export default TaskDashboard;
```

### AI Analytics Hook

```javascript
// src/hooks/useAIAnalytics.js
import { useState, useEffect } from 'react';
import axios from 'axios';

export const useAIAnalytics = (userId) => {
  const [analytics, setAnalytics] = useState({
    riskScore: null,
    burnoutScore: null,
    anomalies: [],
    loading: true
  });

  useEffect(() => {
    if (userId) {
      fetchAnalytics();
    }
  }, [userId]);

  const fetchAnalytics = async () => {
    try {
      const [riskRes, burnoutRes, anomalyRes] = await Promise.all([
        axios.post(`${process.env.REACT_APP_ML_API_URL}/api/ml/risk-detection`, {
          userId
        }),
        axios.post(`${process.env.REACT_APP_ML_API_URL}/api/ml/burnout-detection`, {
          userId
        }),
        axios.get(`${process.env.REACT_APP_ML_API_URL}/api/ml/anomalies/${userId}`)
      ]);

      setAnalytics({
        riskScore: riskRes.data.riskScore,
        burnoutScore: burnoutRes.data.burnoutScore,
        anomalies: anomalyRes.data,
        loading: false
      });
    } catch (error) {
      console.error('Error fetching analytics:', error);
      setAnalytics(prev => ({ ...prev, loading: false }));
    }
  };

  return { analytics, refetchAnalytics: fetchAnalytics };
};
```

## Backend Implementation Patterns

### User Model (MongoDB/Mongoose)

```javascript
// backend/models/User.js
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');

const UserSchema = new mongoose.Schema({
  name: {
    type: String,
    required: true
  },
  email: {
    type: String,
    required: true,
    unique: true,
    lowercase: true
  },
  password: {
    type: String,
    required: true,
    select: false
  },
  role: {
    type: String,
    enum: ['user', 'admin'],
    default: 'user'
  },
  department: String,
  isActive: {
    type: Boolean,
    default: true
  },
  lastLogin: Date,
  failedLoginAttempts: {
    type: Number,
    default: 0
  },
  createdAt: {
    type: Date,
    default: Date.now
  }
});

UserSchema.pre('save', async function(next) {
  if (!this.isModified('password')) return next();
  this.password = await bcrypt.hash(this.password, 12);
  next();
});

UserSchema.methods.comparePassword = async function(candidatePassword) {
  return await bcrypt.compare(candidatePassword, this.password);
};

module.exports = mongoose.model('User', UserSchema);
```

### Auth Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

exports.protect = async (req, res, next) => {
  let token;

  if (req.headers.authorization && req.headers.authorization.startsWith('Bearer')) {
    token = req.headers.authorization.split(' ')[1];
  }

  if (!token) {
    return res.status(401).json({ message: 'Not authorized' });
  }

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = await User.findById(decoded.id);
    next();
  } catch (error) {
    return res.status(401).json({ message: 'Invalid token' });
  }
};

exports.restrictTo = (...roles) => {
  return (req, res, next) => {
    if (!roles.includes(req.user.role)) {
      return res.status(403).json({ message: 'Forbidden' });
    }
    next();
  };
};
```

### Task Controller

```javascript
// backend/controllers/taskController.js
const Task = require('../models/Task');

exports.createTask = async (req, res) => {
  try {
    const task = await Task.create({
      ...req.body,
      createdBy: req.user.id
    });
    res.status(201).json(task);
  } catch (error) {
    res.status(400).json({ message: error.message });
  }
};

exports.getUserTasks = async (req, res) => {
  try {
    const tasks = await Task.find({ assignedTo: req.params.userId })
      .populate('assignedTo', 'name email')
      .populate('createdBy', 'name');
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
};

exports.updateTaskStatus = async (req, res) => {
  try {
    const task = await Task.findByIdAndUpdate(
      req.params.id,
      { status: req.body.status, updatedAt: Date.now() },
      { new: true, runValidators: true }
    );
    if (!task) {
      return res.status(404).json({ message: 'Task not found' });
    }
    res.json(task);
  } catch (error) {
    res.status(400).json({ message: error.message });
  }
};
```

## ML Service Implementation

### Risk Detection Model

```python
# ml-service/models/risk_detector.py
from sklearn.ensemble import RandomForestClassifier
import numpy as np
import joblib
import os

class RiskDetector:
    def __init__(self, model_path='./models/risk_model.pkl'):
        self.model_path = model_path
        if os.path.exists(model_path):
            self.model = joblib.load(model_path)
        else:
            self.model = RandomForestClassifier(n_estimators=100, random_state=42)
            self.is_trained = False
    
    def extract_features(self, user_data):
        """Extract features from user data"""
        features = [
            user_data.get('failedLogins', 0),
            user_data.get('taskCompletionRate', 0.0),
            user_data.get('avgResponseTime', 0),
            user_data.get('ticketsRaised', 0),
            user_data.get('missedDeadlines', 0),
            user_data.get('overtimeHours', 0)
        ]
        return np.array(features).reshape(1, -1)
    
    def predict_risk(self, user_data):
        """Predict risk score for a user"""
        features = self.extract_features(user_data)
        
        if hasattr(self.model, 'predict_proba'):
            risk_score = self.model.predict_proba(features)[0][1]
        else:
            risk_score = 0.5  # Default if model not trained
        
        risk_level = 'low' if risk_score < 0.3 else 'medium' if risk_score < 0.7 else 'high'
        
        factors = []
        if user_data.get('failedLogins', 0) > 2:
            factors.append('Multiple failed login attempts')
        if user_data.get('taskCompletionRate', 1.0) < 0.7:
            factors.append('Low task completion rate')
        if user_data.get('missedDeadlines', 0) > 3:
            factors.append('Frequent missed deadlines')
        
        return {
            'riskScore': float(risk_score),
            'riskLevel': risk_level,
            'factors': factors
        }
    
    def train(self, X, y):
        """Train the model"""
        self.model.fit(X, y)
        joblib.dump(self.model, self.model_path)
        self.is_trained = True
```

### FastAPI Main Service

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from models.risk_detector import RiskDetector
from models.burnout_detector import BurnoutDetector
from models.ticket_classifier import TicketClassifier
import uvicorn

app = FastAPI(title="Enterprise ML Service")

risk_detector = RiskDetector()
burnout_detector = BurnoutDetector()
ticket_classifier = TicketClassifier()

class RiskRequest(BaseModel):
    userId: str
    failedLogins: int = 0
    taskCompletionRate: float = 1.0
    avgResponseTime: int = 0
    ticketsRaised: int = 0
    missedDeadlines: int = 0
    overtimeHours: int = 0

class BurnoutRequest(BaseModel):
    userId: str
    weeklyHours: int
    overtimeHours: int
    taskLoad: int
    missedDeadlines: int
    stressLevel: int

class TicketRequest(BaseModel):
    title: str
    description: str
    text: str

@app.post("/api/ml/risk-detection")
async def detect_risk(request: RiskRequest):
    try:
        result = risk_detector.predict_risk(request.dict())
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/burnout-detection")
async def detect_burnout(request: BurnoutRequest):
    try:
        result = burnout_detector.analyze(request.dict())
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/classify-ticket")
async def classify_ticket(request: TicketRequest):
    try:
        result = ticket_classifier.classify(request.dict())
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "ml-analytics"}

if __name__ == "__main__":
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

## Configuration

### Backend Environment Variables

```bash
# backend/.env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
JWT_SECRET=your_super_secret_jwt_key_here
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=production
CORS_ORIGIN=http://localhost:3000
RATE_LIMIT_MAX=100
RATE_LIMIT_WINDOW=15
```

### ML Service Configuration

```python
# ml-service/config.py
import os
from pydantic import BaseSettings

class Settings(BaseSettings):
    mongodb_uri: str = os.getenv('MONGODB_URI', 'mongodb://localhost:27017/enterprise_user_mgmt')
    model_path: str = os.getenv('MODEL_PATH', './models')
    log_level: str = os.getenv('LOG_LEVEL', 'INFO')
    risk_threshold: float = 0.7
    burnout_threshold: float = 0.75
    
    class Config:
        env_file = '.env'

settings = Settings()
```

## Common Troubleshooting

### JWT Token Issues

```javascript
// Check token expiration
const isTokenExpired = (token) => {
  const decoded = jwt.decode(token);
  if (!decoded || !decoded.exp) return true;
  return decoded.exp < Date.now() / 1000;
};

// Refresh token if needed
if (isTokenExpired(token)) {
  // Redirect to login or refresh token
  await refreshToken();
}
```

### MongoDB Connection Issues

```javascript
// backend/config/db.js
const mongoose = require('mongoose');

const connectDB = async () => {
  try {
    await mongoose.connect(process.env.MONGODB_URI, {
      useNewUrlParser: true,
      useUnifiedTopology: true,
      serverSelectionTimeoutMS: 5000,
      socketTimeoutMS: 45000,
    });
    console.log('MongoDB connected');
  } catch (error) {
    console.error('MongoDB connection error:', error);
    process.exit(1);
  }
};

module.exports = connectDB;
```

### ML Model Not Loading

```python
# ml-service/models/base_model.py
import os
import logging

logger = logging.getLogger(__name__)

def safe_load_model(model_path, default_model):
    """Safely load model with fallback"""
    try:
        if os.path.exists(model_path):
            model = joblib.load(model_path)
            logger.info(f"Model loaded from {model_path}")
            return model
        else:
            logger.warning(f"Model not found at {model_path}, using default")
            return default_model
    except Exception as e:
        logger.error(f"Error loading model: {e}")
        return default_model
```

### CORS Issues

```javascript
// backend/server.js
const cors = require('cors');

app.use(cors({
  origin: process.env.CORS_ORIGIN || 'http://localhost:3000',
  credentials: true,
  methods: ['GET', 'POST', 'PUT', 'DELETE', 'PATCH'],
  allowedHeaders: ['Content-Type', 'Authorization']
}));
```

## Deployment

### Production Build

```bash
# Build frontend
cd frontend
npm run build

# Start backend in production
cd backend
NODE_ENV=production npm start

# Start ML service
cd ml-service
gunicorn main:app --workers 4 --worker-class uvicorn.workers.UvicornWorker --bind 0.0.0.0:8000
```

### Docker Deployment

```dockerfile
# backend/Dockerfile
FROM node:16-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
EXPOSE 5000
CMD ["npm", "start"]
```

```dockerfile
# ml-service/Dockerfile
FROM python:3.9-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 8000
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Docker Compose

```yaml
# docker-compose.yml
version: '3.8'

services:
  mongodb:
    image: mongo:5
    ports:
      - "27017:27017"
    volumes:
      - mongo_data:/data/db

  backend:
    build: ./backend
    ports:
      - "5000:5000"
    environment:
      - MONGODB_URI=mongodb://mongodb:27017/enterprise_user_mgmt
      - JWT_SECRET=${JWT_SECRET}
    depends_on:
      - mongodb

  ml-service:
    build: ./ml-service
    ports:
      - "8000:8000"
    environment:
      - MONGODB_URI=mongodb://mongodb:27017/enterprise_user_mgmt
    depends_on:
      - mongodb

  frontend:
    build: ./frontend
    ports:
      - "3000:3000"
    environment:
      - REACT_APP_API_URL=http://backend:5000
      - REACT_APP_ML_API_URL=http://ml-service:8000
    depends_on:
      - backend
      - ml-service

volumes:
  mongo_data:
```

## Key Patterns

### Real-time Updates with WebSocket

```javascript
// backend/socket.js
const socketIO = require('socket.io');

const initializeSocket = (server) => {
  const io = socketIO(server, {
    cors: { origin: process.env.CORS_ORIGIN }
  });

  io.on('connection', (socket) => {
    console.log('Client connected:', socket.id);

    socket.on('join-dashboard', (userId) => {
      socket.join(`user-${userId}`);
    });

    socket.on('disconnect', () => {
      console.log('Client disconnected:', socket.id);
    });
  });

  return io;
};

// Emit event when task updated
const notifyTaskUpdate = (io, userId, task) => {
  io.to(`user-${userId}`).emit('task-updated', task);
};

module.exports = { initializeSocket, notifyTaskUpdate };
```

### Batch Processing for Analytics

```python
# ml-service/batch_processor.py
from apscheduler.schedulers.asyncio import AsyncIOScheduler
import asyncio

scheduler = AsyncIOScheduler()

@scheduler.scheduled_job('cron', hour=2)  # Run at 2 AM daily
async def batch_risk_analysis():
    """Run risk analysis for all users"""
    users = await get_all_users()
    for user in users:
        risk_score = risk_detector.predict_risk(user)
        await save_risk_score(user['id'], risk_score)
    print(f"Batch analysis complete for {len(users)} users")

scheduler.start()
```

This skill provides comprehensive coverage of the Enterprise User Management System with AI Analytics, enabling AI agents to assist developers in implementing, configuring, and extending this full-stack application with intelligent features.
