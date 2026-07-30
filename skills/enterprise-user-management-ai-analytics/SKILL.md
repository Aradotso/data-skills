---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics, task tracking, and intelligent ticket routing
triggers:
  - "set up enterprise user management system"
  - "create AI-powered user dashboard"
  - "implement task management with AI analytics"
  - "build user management system with machine learning"
  - "integrate AI ticket classification"
  - "configure burnout detection and risk prediction"
  - "deploy enterprise management system with analytics"
  - "implement JWT authentication for user management"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This is a full-stack enterprise user management platform that combines user administration, task tracking, support tickets, and AI-powered analytics. The system provides risk detection, anomaly detection, burnout analysis, and predictive insights using machine learning models.

## Architecture Overview

The system consists of three main components:
- **Frontend**: React.js application for user and admin interfaces
- **Backend**: Node.js REST API with MongoDB for data persistence
- **ML Service**: FastAPI service with scikit-learn and River for real-time AI predictions

## Installation

### Prerequisites

```bash
# Required tools
node -v  # v14+ required
python -v  # Python 3.8+ required
mongod --version  # MongoDB 4.4+ required
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
JWT_SECRET=your_jwt_secret_here
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
EOF

npm start
# Backend runs at http://localhost:5000
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt

# Create .env file
cat > .env << EOF
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
MODEL_PATH=./models
LOG_LEVEL=info
EOF

uvicorn main:app --reload
# ML service runs at http://localhost:8000
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

npm start
# Frontend runs at http://localhost:3000
```

## Key API Endpoints

### Authentication

```javascript
// User login
POST /api/auth/login
Content-Type: application/json

{
  "email": "user@example.com",
  "password": "password123"
}

// Response
{
  "token": "jwt_token_here",
  "user": {
    "id": "userId",
    "name": "User Name",
    "role": "user" | "admin"
  }
}
```

### User Management (Admin)

```javascript
// Get all users
GET /api/users
Authorization: Bearer {token}

// Create new user
POST /api/users
Authorization: Bearer {token}
Content-Type: application/json

{
  "name": "John Doe",
  "email": "john@example.com",
  "role": "user",
  "department": "Engineering"
}

// Update user
PUT /api/users/:userId
Authorization: Bearer {token}

// Delete user
DELETE /api/users/:userId
Authorization: Bearer {token}
```

### Task Management

```javascript
// Get user tasks
GET /api/tasks?userId={userId}
Authorization: Bearer {token}

// Create task
POST /api/tasks
Authorization: Bearer {token}
Content-Type: application/json

{
  "title": "Complete project documentation",
  "description": "Write comprehensive docs",
  "assignedTo": "userId",
  "priority": "high",
  "dueDate": "2026-05-01",
  "status": "todo"
}

// Update task status
PATCH /api/tasks/:taskId
Authorization: Bearer {token}

{
  "status": "in-progress" | "done",
  "timeSpent": 120  // minutes
}
```

### Support Tickets

```javascript
// Create ticket
POST /api/tickets
Authorization: Bearer {token}

{
  "title": "Login issue",
  "description": "Cannot access dashboard",
  "priority": "medium",
  "category": "technical"
}

// Get user tickets
GET /api/tickets?userId={userId}
Authorization: Bearer {token}

// Admin: Route ticket with AI
POST /api/tickets/:ticketId/classify
Authorization: Bearer {token}
```

### AI Analytics Endpoints

```python
# Risk prediction
POST http://localhost:8000/api/ml/predict-risk
Content-Type: application/json

{
  "user_id": "userId",
  "login_attempts": 5,
  "failed_logins": 2,
  "unusual_hours": 1,
  "multiple_locations": 0
}

# Response
{
  "risk_score": 0.72,
  "risk_level": "high",
  "factors": ["failed_logins", "login_attempts"]
}
```

```python
# Burnout detection
POST http://localhost:8000/api/ml/detect-burnout

{
  "user_id": "userId",
  "tasks_completed": 45,
  "avg_task_duration": 180,
  "overtime_hours": 15,
  "missed_deadlines": 3
}

# Response
{
  "burnout_probability": 0.68,
  "recommendation": "Consider workload redistribution"
}
```

```python
# Ticket classification
POST http://localhost:8000/api/ml/classify-ticket

{
  "title": "Cannot access reporting module",
  "description": "Error 403 when clicking reports",
  "priority": "high"
}

# Response
{
  "category": "technical",
  "suggested_team": "engineering",
  "confidence": 0.89
}
```

## Frontend Integration

### React Authentication Hook

```javascript
// src/hooks/useAuth.js
import { useState, useEffect } from 'react';
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL;

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
      const { data } = await axios.get(`${API_URL}/api/auth/me`);
      setUser(data.user);
    } catch (error) {
      localStorage.removeItem('token');
      delete axios.defaults.headers.common['Authorization'];
    } finally {
      setLoading(false);
    }
  };

  const login = async (email, password) => {
    const { data } = await axios.post(`${API_URL}/api/auth/login`, {
      email,
      password
    });
    localStorage.setItem('token', data.token);
    axios.defaults.headers.common['Authorization'] = `Bearer ${data.token}`;
    setUser(data.user);
    return data;
  };

  const logout = () => {
    localStorage.removeItem('token');
    delete axios.defaults.headers.common['Authorization'];
    setUser(null);
  };

  return { user, loading, login, logout };
};
```

### Task Management Component

```javascript
// src/components/TaskBoard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL;

const TaskBoard = ({ userId }) => {
  const [tasks, setTasks] = useState({
    todo: [],
    'in-progress': [],
    done: []
  });

  useEffect(() => {
    fetchTasks();
  }, [userId]);

  const fetchTasks = async () => {
    try {
      const { data } = await axios.get(`${API_URL}/api/tasks?userId=${userId}`);
      const grouped = data.reduce((acc, task) => {
        acc[task.status].push(task);
        return acc;
      }, { todo: [], 'in-progress': [], done: [] });
      setTasks(grouped);
    } catch (error) {
      console.error('Failed to fetch tasks:', error);
    }
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    try {
      await axios.patch(`${API_URL}/api/tasks/${taskId}`, {
        status: newStatus
      });
      fetchTasks();
    } catch (error) {
      console.error('Failed to update task:', error);
    }
  };

  const onDragStart = (e, taskId) => {
    e.dataTransfer.setData('taskId', taskId);
  };

  const onDrop = (e, status) => {
    const taskId = e.dataTransfer.getData('taskId');
    updateTaskStatus(taskId, status);
  };

  return (
    <div className="task-board">
      {['todo', 'in-progress', 'done'].map(status => (
        <div
          key={status}
          className="task-column"
          onDrop={(e) => onDrop(e, status)}
          onDragOver={(e) => e.preventDefault()}
        >
          <h3>{status.toUpperCase()}</h3>
          {tasks[status].map(task => (
            <div
              key={task.id}
              draggable
              onDragStart={(e) => onDragStart(e, task.id)}
              className="task-card"
            >
              <h4>{task.title}</h4>
              <p>{task.description}</p>
              <span className={`priority ${task.priority}`}>
                {task.priority}
              </span>
            </div>
          ))}
        </div>
      ))}
    </div>
  );
};

export default TaskBoard;
```

### AI Analytics Dashboard

```javascript
// src/components/AIAnalytics.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const ML_API_URL = process.env.REACT_APP_ML_API_URL;

const AIAnalytics = ({ userId }) => {
  const [analytics, setAnalytics] = useState({
    riskScore: null,
    burnoutProbability: null,
    anomalies: []
  });

  useEffect(() => {
    fetchAnalytics();
  }, [userId]);

  const fetchAnalytics = async () => {
    try {
      // Fetch user activity data
      const activityResponse = await axios.get(
        `${process.env.REACT_APP_API_URL}/api/users/${userId}/activity`
      );
      const activity = activityResponse.data;

      // Get risk prediction
      const riskResponse = await axios.post(
        `${ML_API_URL}/api/ml/predict-risk`,
        {
          user_id: userId,
          login_attempts: activity.loginAttempts,
          failed_logins: activity.failedLogins,
          unusual_hours: activity.unusualHours,
          multiple_locations: activity.multipleLocations
        }
      );

      // Get burnout detection
      const burnoutResponse = await axios.post(
        `${ML_API_URL}/api/ml/detect-burnout`,
        {
          user_id: userId,
          tasks_completed: activity.tasksCompleted,
          avg_task_duration: activity.avgTaskDuration,
          overtime_hours: activity.overtimeHours,
          missed_deadlines: activity.missedDeadlines
        }
      );

      setAnalytics({
        riskScore: riskResponse.data.risk_score,
        burnoutProbability: burnoutResponse.data.burnout_probability,
        anomalies: activity.anomalies || []
      });
    } catch (error) {
      console.error('Failed to fetch analytics:', error);
    }
  };

  return (
    <div className="ai-analytics">
      <h2>AI-Powered Insights</h2>
      
      <div className="metric-card">
        <h3>Security Risk</h3>
        <div className={`score ${analytics.riskScore > 0.7 ? 'high' : 'low'}`}>
          {(analytics.riskScore * 100).toFixed(1)}%
        </div>
      </div>

      <div className="metric-card">
        <h3>Burnout Risk</h3>
        <div className={`score ${analytics.burnoutProbability > 0.6 ? 'high' : 'low'}`}>
          {(analytics.burnoutProbability * 100).toFixed(1)}%
        </div>
      </div>

      {analytics.anomalies.length > 0 && (
        <div className="anomalies">
          <h3>Detected Anomalies</h3>
          <ul>
            {analytics.anomalies.map((anomaly, idx) => (
              <li key={idx}>{anomaly.description}</li>
            ))}
          </ul>
        </div>
      )}
    </div>
  );
};

export default AIAnalytics;
```

## Backend Implementation

### User Model (MongoDB/Mongoose)

```javascript
// backend/models/User.js
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');

const userSchema = new mongoose.Schema({
  name: {
    type: String,
    required: true,
    trim: true
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
    minlength: 6
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
  createdAt: {
    type: Date,
    default: Date.now
  }
});

// Hash password before saving
userSchema.pre('save', async function(next) {
  if (!this.isModified('password')) return next();
  this.password = await bcrypt.hash(this.password, 10);
  next();
});

// Compare password method
userSchema.methods.comparePassword = async function(candidatePassword) {
  return bcrypt.compare(candidatePassword, this.password);
};

module.exports = mongoose.model('User', userSchema);
```

### Authentication Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

const protect = async (req, res, next) => {
  try {
    let token;
    
    if (req.headers.authorization?.startsWith('Bearer')) {
      token = req.headers.authorization.split(' ')[1];
    }

    if (!token) {
      return res.status(401).json({ message: 'Not authorized' });
    }

    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = await User.findById(decoded.id).select('-password');
    
    if (!req.user) {
      return res.status(401).json({ message: 'User not found' });
    }

    next();
  } catch (error) {
    res.status(401).json({ message: 'Not authorized' });
  }
};

const adminOnly = (req, res, next) => {
  if (req.user && req.user.role === 'admin') {
    next();
  } else {
    res.status(403).json({ message: 'Admin access required' });
  }
};

module.exports = { protect, adminOnly };
```

### Task Controller

```javascript
// backend/controllers/taskController.js
const Task = require('../models/Task');
const axios = require('axios');

exports.getTasks = async (req, res) => {
  try {
    const { userId } = req.query;
    const query = userId ? { assignedTo: userId } : {};
    
    const tasks = await Task.find(query)
      .populate('assignedTo', 'name email')
      .sort({ createdAt: -1 });
    
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
};

exports.createTask = async (req, res) => {
  try {
    const task = new Task({
      ...req.body,
      createdBy: req.user._id
    });
    
    await task.save();
    
    // Get AI prediction for task completion time
    try {
      const prediction = await axios.post(
        `${process.env.ML_SERVICE_URL}/api/ml/predict-task-duration`,
        {
          title: task.title,
          priority: task.priority,
          assignedTo: task.assignedTo
        }
      );
      
      task.estimatedDuration = prediction.data.duration;
      await task.save();
    } catch (mlError) {
      console.error('ML prediction failed:', mlError);
    }
    
    res.status(201).json(task);
  } catch (error) {
    res.status(400).json({ message: error.message });
  }
};

exports.updateTask = async (req, res) => {
  try {
    const task = await Task.findByIdAndUpdate(
      req.params.id,
      req.body,
      { new: true }
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

### FastAPI Main Application

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import Optional, List
import numpy as np
from sklearn.ensemble import RandomForestClassifier
from river import anomaly, ensemble
import joblib
import os

app = FastAPI(title="Enterprise User Management ML Service")

# Load or initialize models
MODEL_PATH = os.getenv('MODEL_PATH', './models')
risk_model = None
burnout_model = None
anomaly_detector = anomaly.HalfSpaceTrees(n_trees=10, seed=42)

class RiskPredictionRequest(BaseModel):
    user_id: str
    login_attempts: int
    failed_logins: int
    unusual_hours: int
    multiple_locations: int

class BurnoutDetectionRequest(BaseModel):
    user_id: str
    tasks_completed: int
    avg_task_duration: float
    overtime_hours: float
    missed_deadlines: int

class TicketClassificationRequest(BaseModel):
    title: str
    description: str
    priority: str

@app.on_event("startup")
async def load_models():
    global risk_model, burnout_model
    try:
        risk_model = joblib.load(f"{MODEL_PATH}/risk_model.pkl")
        burnout_model = joblib.load(f"{MODEL_PATH}/burnout_model.pkl")
    except FileNotFoundError:
        # Initialize with default models if not found
        risk_model = RandomForestClassifier(n_estimators=100, random_state=42)
        burnout_model = RandomForestClassifier(n_estimators=100, random_state=42)

@app.post("/api/ml/predict-risk")
async def predict_risk(request: RiskPredictionRequest):
    try:
        features = np.array([[
            request.login_attempts,
            request.failed_logins,
            request.unusual_hours,
            request.multiple_locations
        ]])
        
        # Simple rule-based scoring if model not trained
        risk_score = (
            (request.failed_logins / max(request.login_attempts, 1)) * 0.4 +
            (request.unusual_hours / 10) * 0.3 +
            (request.multiple_locations / 5) * 0.3
        )
        
        risk_level = "low"
        if risk_score > 0.7:
            risk_level = "high"
        elif risk_score > 0.4:
            risk_level = "medium"
        
        factors = []
        if request.failed_logins > 2:
            factors.append("failed_logins")
        if request.unusual_hours > 5:
            factors.append("unusual_hours")
        if request.multiple_locations > 2:
            factors.append("multiple_locations")
        
        return {
            "risk_score": float(risk_score),
            "risk_level": risk_level,
            "factors": factors
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/detect-burnout")
async def detect_burnout(request: BurnoutDetectionRequest):
    try:
        # Normalized burnout score
        workload_score = min(request.tasks_completed / 50, 1.0)
        duration_score = min(request.avg_task_duration / 240, 1.0)
        overtime_score = min(request.overtime_hours / 20, 1.0)
        deadline_score = min(request.missed_deadlines / 5, 1.0)
        
        burnout_probability = (
            workload_score * 0.3 +
            duration_score * 0.2 +
            overtime_score * 0.3 +
            deadline_score * 0.2
        )
        
        recommendation = "Workload is healthy"
        if burnout_probability > 0.7:
            recommendation = "High burnout risk - immediate intervention needed"
        elif burnout_probability > 0.5:
            recommendation = "Consider workload redistribution"
        
        return {
            "burnout_probability": float(burnout_probability),
            "recommendation": recommendation,
            "factors": {
                "workload": float(workload_score),
                "task_duration": float(duration_score),
                "overtime": float(overtime_score),
                "missed_deadlines": float(deadline_score)
            }
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/classify-ticket")
async def classify_ticket(request: TicketClassificationRequest):
    try:
        text = f"{request.title} {request.description}".lower()
        
        # Simple keyword-based classification
        categories = {
            "technical": ["error", "bug", "crash", "403", "500", "access", "login"],
            "billing": ["payment", "invoice", "charge", "subscription"],
            "feature": ["request", "add", "new", "enhancement"],
            "support": ["help", "how to", "question", "guide"]
        }
        
        scores = {}
        for category, keywords in categories.items():
            score = sum(1 for keyword in keywords if keyword in text)
            scores[category] = score
        
        predicted_category = max(scores, key=scores.get)
        confidence = scores[predicted_category] / (sum(scores.values()) or 1)
        
        team_mapping = {
            "technical": "engineering",
            "billing": "finance",
            "feature": "product",
            "support": "customer_success"
        }
        
        return {
            "category": predicted_category,
            "suggested_team": team_mapping.get(predicted_category, "support"),
            "confidence": float(confidence),
            "all_scores": scores
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/detect-anomaly")
async def detect_anomaly(user_activity: dict):
    try:
        features = {
            'login_time': user_activity.get('login_time', 0),
            'session_duration': user_activity.get('session_duration', 0),
            'actions_count': user_activity.get('actions_count', 0)
        }
        
        score = anomaly_detector.score_one(features)
        anomaly_detector.learn_one(features)
        
        is_anomaly = score > 0.7
        
        return {
            "is_anomaly": is_anomaly,
            "anomaly_score": float(score),
            "threshold": 0.7
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "ml-service"}
```

## Common Patterns

### Role-Based Access Control

```javascript
// Protect admin routes
const express = require('express');
const router = express.Router();
const { protect, adminOnly } = require('../middleware/auth');

router.get('/users', protect, adminOnly, getAllUsers);
router.post('/users', protect, adminOnly, createUser);
router.delete('/users/:id', protect, adminOnly, deleteUser);

// User can only access their own data
router.get('/tasks/me', protect, getMyTasks);
router.get('/profile', protect, getProfile);
```

### Real-time Updates with WebSocket

```javascript
// backend/websocket.js
const WebSocket = require('ws');

const setupWebSocket = (server) => {
  const wss = new WebSocket.Server({ server });

  wss.on('connection', (ws) => {
    ws.on('message', (message) => {
      const data = JSON.parse(message);
      
      // Broadcast to all connected clients
      wss.clients.forEach((client) => {
        if (client.readyState === WebSocket.OPEN) {
          client.send(JSON.stringify({
            type: data.type,
            payload: data.payload
          }));
        }
      });
    });
  });

  return wss;
};

module.exports = setupWebSocket;
```

## Troubleshooting

### MongoDB Connection Issues

```bash
# Check MongoDB is running
sudo systemctl status mongod

# Restart MongoDB
sudo systemctl restart mongod

# Check connection string in .env
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
```

### JWT Authentication Errors

```javascript
// Verify token format
const token = req.headers.authorization?.split(' ')[1];
console.log('Token:', token);

// Check JWT_SECRET is set
if (!process.env.JWT_SECRET) {
  throw new Error('JWT_SECRET not configured');
}

// Clear token and re-login
localStorage.removeItem('token');
```

### ML Service Not Responding

```bash
# Check ML service logs
cd ml-service
uvicorn main:app --reload --log-level debug

# Verify Python dependencies
pip list | grep -E "fastapi|sklearn|river"

# Test health endpoint
curl http://localhost:8000/health
```

### CORS Issues

```javascript
// backend/server.js
const cors = require('cors');

app.use(cors({
  origin: process.env.FRONTEND_URL || 'http://localhost:3000',
  credentials: true
}));
```

### Task Board Not Updating

```javascript
// Force re-fetch after update
const updateTaskStatus = async (taskId, newStatus) => {
  await axios.patch(`${API_URL}/api/tasks/${taskId}`, { status: newStatus });
  await fetchTasks(); // Refresh task list
};

// Check WebSocket connection
const ws = new WebSocket('ws://localhost:5000');
ws.onmessage = (event) => {
  const data = JSON.parse(event.data);
  if (data.type === 'TASK_UPDATED') {
    fetchTasks();
  }
};
```

## Performance Optimization

### Backend Caching

```javascript
const NodeCache = require('node-cache');
const cache = new NodeCache({ stdTTL: 600 }); // 10 min cache

exports.getAnalytics = async (req, res) => {
  const cacheKey = `analytics_${req.user.id}`;
  const cached = cache.get(cacheKey);
  
  if (cached) {
    return res.json(cached);
  }
  
  const analytics = await calculateAnalytics(req.user.id);
  cache.set(cacheKey, analytics);
  res.json(analytics);
};
```

### Database Indexing

```javascript
// Add indexes for faster queries
userSchema.index({ email: 1 });
taskSchema.index({ assignedTo: 1, status: 1 });
ticketSchema.index({ createdAt: -1 });
```

This system provides a complete enterprise user management solution with AI-powered insights for security, productivity, and organizational efficiency.
