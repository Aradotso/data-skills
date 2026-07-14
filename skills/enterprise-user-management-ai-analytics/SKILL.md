---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for risk detection, anomaly detection, and workload insights
triggers:
  - "set up enterprise user management system"
  - "integrate AI analytics for user management"
  - "implement user task tracking with AI insights"
  - "create admin dashboard with ML predictions"
  - "build user management with anomaly detection"
  - "add AI-based ticket classification system"
  - "implement burnout detection for users"
  - "set up Kanban board with time tracking"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A full-stack enterprise user management platform that combines user/task management with AI-powered analytics including risk detection, anomaly detection, burnout analysis, and predictive project insights. Built with React, Node.js/Spring Boot, MongoDB, and FastAPI ML service.

## What This Project Does

This system provides:
- **User Management**: Role-based access control (Admin/User) with JWT authentication
- **Task Management**: Kanban board (To Do → In Progress → Done) with time tracking
- **Support Tickets**: AI-powered ticket classification and routing
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, project delay prediction
- **Admin Dashboard**: Organization-wide analytics, audit logs, user monitoring
- **User Dashboard**: Personal task overview, performance insights, notifications

## Installation

### Prerequisites
- Node.js 14+ and npm
- Python 3.8+
- MongoDB instance (local or cloud)

### 1. Clone and Setup

```bash
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics
```

### 2. Backend Setup

```bash
cd backend
npm install
```

Create `backend/.env`:
```env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise-ums
JWT_SECRET=your_jwt_secret_key
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
```

Start backend:
```bash
npm start
```

### 3. ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt
```

Create `ml-service/.env`:
```env
MONGODB_URI=mongodb://localhost:27017/enterprise-ums
MODEL_PATH=./models
LOG_LEVEL=INFO
```

Start ML service:
```bash
uvicorn main:app --reload --port 8000
```

### 4. Frontend Setup

```bash
cd frontend
npm install
```

Create `frontend/.env`:
```env
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
```

Start frontend:
```bash
npm start
```

## Core API Endpoints

### Authentication (Backend)

```javascript
// User login
POST /api/auth/login
{
  "email": "user@example.com",
  "password": "password123"
}

// Response includes JWT token
{
  "token": "eyJhbGciOiJIUzI1NiIs...",
  "user": {
    "id": "user_id",
    "email": "user@example.com",
    "role": "user"
  }
}
```

### User Management (Backend)

```javascript
// Get all users (Admin only)
GET /api/users
Headers: { "Authorization": "Bearer <token>" }

// Create user (Admin)
POST /api/users
{
  "name": "John Doe",
  "email": "john@example.com",
  "role": "user",
  "department": "Engineering"
}

// Update user
PUT /api/users/:userId
{
  "name": "John Smith",
  "department": "DevOps"
}

// Delete user
DELETE /api/users/:userId
```

### Task Management (Backend)

```javascript
// Get user tasks
GET /api/tasks?userId=<userId>

// Create task
POST /api/tasks
{
  "title": "Implement feature X",
  "description": "Add new authentication flow",
  "assignedTo": "user_id",
  "priority": "high",
  "dueDate": "2026-05-01",
  "status": "todo"
}

// Update task status
PATCH /api/tasks/:taskId/status
{
  "status": "in-progress"
}

// Track time
POST /api/tasks/:taskId/time
{
  "timeSpent": 3600 // seconds
}
```

### Support Tickets (Backend)

```javascript
// Create ticket
POST /api/tickets
{
  "title": "Cannot access dashboard",
  "description": "Getting 404 error",
  "priority": "medium",
  "category": "technical"
}

// Get tickets
GET /api/tickets?status=open

// Update ticket
PATCH /api/tickets/:ticketId
{
  "status": "resolved",
  "resolution": "Fixed authentication issue"
}
```

## AI/ML Service API

### Risk Prediction

```python
# Endpoint: POST /api/ml/predict-risk
# Analyzes user behavior patterns for risk assessment

import requests

response = requests.post(
    "http://localhost:8000/api/ml/predict-risk",
    json={
        "userId": "user_123",
        "loginFrequency": 45,
        "taskCompletionRate": 0.72,
        "avgResponseTime": 2.5,
        "ticketsRaised": 8,
        "failedLoginAttempts": 2
    }
)

result = response.json()
# {
#   "riskScore": 0.35,
#   "riskLevel": "medium",
#   "factors": ["high_ticket_volume", "failed_logins"]
# }
```

### Anomaly Detection

```python
# Endpoint: POST /api/ml/detect-anomaly
# Detects unusual patterns in user activity

response = requests.post(
    "http://localhost:8000/api/ml/detect-anomaly",
    json={
        "userId": "user_123",
        "loginTime": "2026-04-15T03:30:00Z",
        "loginLocation": "Unknown City",
        "activityPattern": [0, 0, 0, 1, 0, 0, 0]
    }
)

result = response.json()
# {
#   "isAnomaly": true,
#   "anomalyScore": 0.89,
#   "reasons": ["unusual_login_time", "new_location"]
# }
```

### Burnout Detection

```python
# Endpoint: POST /api/ml/detect-burnout
# Analyzes workload for burnout risk

response = requests.post(
    "http://localhost:8000/api/ml/detect-burnout",
    json={
        "userId": "user_123",
        "tasksAssigned": 25,
        "tasksCompleted": 18,
        "avgWorkHours": 11.5,
        "weekendWork": 4,
        "overtimeDays": 12
    }
)

result = response.json()
# {
#   "burnoutRisk": "high",
#   "burnoutScore": 0.78,
#   "recommendations": [
#     "Reduce task assignment by 30%",
#     "Enforce weekend rest"
#   ]
# }
```

### Ticket Classification

```python
# Endpoint: POST /api/ml/classify-ticket
# Auto-classifies and routes support tickets

response = requests.post(
    "http://localhost:8000/api/ml/classify-ticket",
    json={
        "title": "Cannot reset password",
        "description": "Reset link not working after multiple attempts"
    }
)

result = response.json()
# {
#   "category": "authentication",
#   "priority": "high",
#   "suggestedTeam": "security",
#   "confidence": 0.92
# }
```

## Frontend Integration Patterns

### Authentication Hook

```javascript
// src/hooks/useAuth.js
import { useState, useEffect } from 'react';
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL;

export const useAuth = () => {
  const [user, setUser] = useState(null);
  const [token, setToken] = useState(localStorage.getItem('token'));

  useEffect(() => {
    if (token) {
      axios.defaults.headers.common['Authorization'] = `Bearer ${token}`;
      fetchUserProfile();
    }
  }, [token]);

  const login = async (email, password) => {
    try {
      const response = await axios.post(`${API_URL}/api/auth/login`, {
        email,
        password
      });
      
      setToken(response.data.token);
      setUser(response.data.user);
      localStorage.setItem('token', response.data.token);
      
      return { success: true };
    } catch (error) {
      return { success: false, error: error.response?.data?.message };
    }
  };

  const logout = () => {
    setToken(null);
    setUser(null);
    localStorage.removeItem('token');
    delete axios.defaults.headers.common['Authorization'];
  };

  const fetchUserProfile = async () => {
    try {
      const response = await axios.get(`${API_URL}/api/auth/profile`);
      setUser(response.data);
    } catch (error) {
      logout();
    }
  };

  return { user, login, logout, isAuthenticated: !!token };
};
```

### Task Management Component

```javascript
// src/components/KanbanBoard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL;

const KanbanBoard = ({ userId }) => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });

  useEffect(() => {
    fetchTasks();
  }, [userId]);

  const fetchTasks = async () => {
    try {
      const response = await axios.get(`${API_URL}/api/tasks?userId=${userId}`);
      const tasksByStatus = {
        todo: response.data.filter(t => t.status === 'todo'),
        inProgress: response.data.filter(t => t.status === 'in-progress'),
        done: response.data.filter(t => t.status === 'done')
      };
      setTasks(tasksByStatus);
    } catch (error) {
      console.error('Failed to fetch tasks:', error);
    }
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    try {
      await axios.patch(`${API_URL}/api/tasks/${taskId}/status`, {
        status: newStatus
      });
      fetchTasks();
    } catch (error) {
      console.error('Failed to update task:', error);
    }
  };

  return (
    <div className="kanban-board">
      {['todo', 'inProgress', 'done'].map(status => (
        <div key={status} className="kanban-column">
          <h3>{status.replace(/([A-Z])/g, ' $1').toUpperCase()}</h3>
          {tasks[status].map(task => (
            <div key={task.id} className="task-card">
              <h4>{task.title}</h4>
              <p>{task.description}</p>
              <select 
                value={task.status} 
                onChange={(e) => updateTaskStatus(task.id, e.target.value)}
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

export default KanbanBoard;
```

### AI Analytics Dashboard

```javascript
// src/components/AIAnalytics.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const ML_API_URL = process.env.REACT_APP_ML_API_URL;

const AIAnalytics = ({ userId }) => {
  const [analytics, setAnalytics] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchAnalytics();
  }, [userId]);

  const fetchAnalytics = async () => {
    setLoading(true);
    try {
      const [riskRes, burnoutRes] = await Promise.all([
        axios.post(`${ML_API_URL}/api/ml/predict-risk`, {
          userId,
          // Include actual user metrics here
        }),
        axios.post(`${ML_API_URL}/api/ml/detect-burnout`, {
          userId,
          // Include workload metrics here
        })
      ]);

      setAnalytics({
        risk: riskRes.data,
        burnout: burnoutRes.data
      });
    } catch (error) {
      console.error('Failed to fetch analytics:', error);
    } finally {
      setLoading(false);
    }
  };

  if (loading) return <div>Loading analytics...</div>;

  return (
    <div className="ai-analytics">
      <div className="metric-card">
        <h3>Risk Assessment</h3>
        <div className={`risk-level ${analytics.risk.riskLevel}`}>
          {analytics.risk.riskLevel.toUpperCase()}
        </div>
        <p>Score: {(analytics.risk.riskScore * 100).toFixed(1)}%</p>
        <ul>
          {analytics.risk.factors.map((factor, i) => (
            <li key={i}>{factor}</li>
          ))}
        </ul>
      </div>

      <div className="metric-card">
        <h3>Burnout Detection</h3>
        <div className={`burnout-risk ${analytics.burnout.burnoutRisk}`}>
          {analytics.burnout.burnoutRisk.toUpperCase()}
        </div>
        <p>Score: {(analytics.burnout.burnoutScore * 100).toFixed(1)}%</p>
        <h4>Recommendations:</h4>
        <ul>
          {analytics.burnout.recommendations.map((rec, i) => (
            <li key={i}>{rec}</li>
          ))}
        </ul>
      </div>
    </div>
  );
};

export default AIAnalytics;
```

## Backend Implementation Patterns

### JWT Authentication Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');

const authenticateToken = (req, res, next) => {
  const authHeader = req.headers['authorization'];
  const token = authHeader && authHeader.split(' ')[1];

  if (!token) {
    return res.status(401).json({ message: 'Access token required' });
  }

  jwt.verify(token, process.env.JWT_SECRET, (err, user) => {
    if (err) {
      return res.status(403).json({ message: 'Invalid or expired token' });
    }
    req.user = user;
    next();
  });
};

const requireAdmin = (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ message: 'Admin access required' });
  }
  next();
};

module.exports = { authenticateToken, requireAdmin };
```

### User Controller

```javascript
// backend/controllers/userController.js
const User = require('../models/User');
const bcrypt = require('bcrypt');

exports.getAllUsers = async (req, res) => {
  try {
    const users = await User.find().select('-password');
    res.json(users);
  } catch (error) {
    res.status(500).json({ message: 'Failed to fetch users', error });
  }
};

exports.createUser = async (req, res) => {
  try {
    const { name, email, password, role, department } = req.body;
    
    const existingUser = await User.findOne({ email });
    if (existingUser) {
      return res.status(400).json({ message: 'User already exists' });
    }

    const hashedPassword = await bcrypt.hash(password, 10);
    
    const user = new User({
      name,
      email,
      password: hashedPassword,
      role: role || 'user',
      department
    });

    await user.save();
    
    const userResponse = user.toObject();
    delete userResponse.password;
    
    res.status(201).json(userResponse);
  } catch (error) {
    res.status(500).json({ message: 'Failed to create user', error });
  }
};

exports.updateUser = async (req, res) => {
  try {
    const { userId } = req.params;
    const updates = req.body;
    
    if (updates.password) {
      updates.password = await bcrypt.hash(updates.password, 10);
    }

    const user = await User.findByIdAndUpdate(
      userId,
      updates,
      { new: true }
    ).select('-password');

    if (!user) {
      return res.status(404).json({ message: 'User not found' });
    }

    res.json(user);
  } catch (error) {
    res.status(500).json({ message: 'Failed to update user', error });
  }
};

exports.deleteUser = async (req, res) => {
  try {
    const { userId } = req.params;
    const user = await User.findByIdAndDelete(userId);

    if (!user) {
      return res.status(404).json({ message: 'User not found' });
    }

    res.json({ message: 'User deleted successfully' });
  } catch (error) {
    res.status(500).json({ message: 'Failed to delete user', error });
  }
};
```

## ML Service Implementation

### Risk Prediction Model

```python
# ml-service/models/risk_predictor.py
from sklearn.ensemble import RandomForestClassifier
import numpy as np
import joblib
import os

class RiskPredictor:
    def __init__(self, model_path='./models/risk_model.pkl'):
        self.model_path = model_path
        self.model = self._load_or_create_model()
        
    def _load_or_create_model(self):
        if os.path.exists(self.model_path):
            return joblib.load(self.model_path)
        else:
            # Create new model
            model = RandomForestClassifier(n_estimators=100, random_state=42)
            return model
    
    def predict_risk(self, features):
        """
        Features: [login_frequency, task_completion_rate, avg_response_time, 
                   tickets_raised, failed_login_attempts]
        """
        X = np.array(features).reshape(1, -1)
        
        if hasattr(self.model, 'predict_proba'):
            risk_score = self.model.predict_proba(X)[0][1]
        else:
            # If model not trained, use rule-based scoring
            risk_score = self._rule_based_risk(features)
        
        risk_level = self._categorize_risk(risk_score)
        factors = self._identify_risk_factors(features, risk_score)
        
        return {
            'riskScore': float(risk_score),
            'riskLevel': risk_level,
            'factors': factors
        }
    
    def _rule_based_risk(self, features):
        # Fallback rule-based scoring
        score = 0.0
        
        # Low login frequency
        if features[0] < 10:
            score += 0.2
        
        # Low task completion
        if features[1] < 0.5:
            score += 0.3
        
        # High ticket volume
        if features[3] > 10:
            score += 0.2
        
        # Failed logins
        if features[4] > 0:
            score += 0.3
        
        return min(score, 1.0)
    
    def _categorize_risk(self, score):
        if score < 0.3:
            return 'low'
        elif score < 0.7:
            return 'medium'
        else:
            return 'high'
    
    def _identify_risk_factors(self, features, score):
        factors = []
        
        if features[0] < 10:
            factors.append('low_activity')
        if features[1] < 0.5:
            factors.append('poor_task_completion')
        if features[3] > 10:
            factors.append('high_ticket_volume')
        if features[4] > 0:
            factors.append('failed_logins')
        
        return factors
    
    def save_model(self):
        os.makedirs(os.path.dirname(self.model_path), exist_ok=True)
        joblib.dump(self.model, self.model_path)
```

### FastAPI Main Application

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from models.risk_predictor import RiskPredictor
from models.burnout_detector import BurnoutDetector
from models.ticket_classifier import TicketClassifier
import os

app = FastAPI(title="Enterprise UMS ML Service")

# Initialize models
risk_predictor = RiskPredictor()
burnout_detector = BurnoutDetector()
ticket_classifier = TicketClassifier()

class RiskPredictionRequest(BaseModel):
    userId: str
    loginFrequency: int
    taskCompletionRate: float
    avgResponseTime: float
    ticketsRaised: int
    failedLoginAttempts: int

class BurnoutDetectionRequest(BaseModel):
    userId: str
    tasksAssigned: int
    tasksCompleted: int
    avgWorkHours: float
    weekendWork: int
    overtimeDays: int

class TicketClassificationRequest(BaseModel):
    title: str
    description: str

@app.post("/api/ml/predict-risk")
async def predict_risk(request: RiskPredictionRequest):
    try:
        features = [
            request.loginFrequency,
            request.taskCompletionRate,
            request.avgResponseTime,
            request.ticketsRaised,
            request.failedLoginAttempts
        ]
        
        result = risk_predictor.predict_risk(features)
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/detect-burnout")
async def detect_burnout(request: BurnoutDetectionRequest):
    try:
        features = [
            request.tasksAssigned,
            request.tasksCompleted,
            request.avgWorkHours,
            request.weekendWork,
            request.overtimeDays
        ]
        
        result = burnout_detector.detect_burnout(features)
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/classify-ticket")
async def classify_ticket(request: TicketClassificationRequest):
    try:
        result = ticket_classifier.classify(
            request.title,
            request.description
        )
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "ml-service"}
```

## Configuration

### MongoDB Schema Examples

```javascript
// backend/models/User.js
const mongoose = require('mongoose');

const userSchema = new mongoose.Schema({
  name: { type: String, required: true },
  email: { type: String, required: true, unique: true },
  password: { type: String, required: true },
  role: { type: String, enum: ['admin', 'user'], default: 'user' },
  department: String,
  createdAt: { type: Date, default: Date.now },
  lastLogin: Date,
  isActive: { type: Boolean, default: true }
});

module.exports = mongoose.model('User', userSchema);
```

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
    enum: ['todo', 'in-progress', 'done'], 
    default: 'todo' 
  },
  priority: { 
    type: String, 
    enum: ['low', 'medium', 'high'], 
    default: 'medium' 
  },
  dueDate: Date,
  timeSpent: { type: Number, default: 0 }, // in seconds
  createdAt: { type: Date, default: Date.now },
  updatedAt: { type: Date, default: Date.now }
});

module.exports = mongoose.model('Task', taskSchema);
```

## Troubleshooting

### Connection Issues

**Problem**: Frontend can't connect to backend

```javascript
// Check CORS configuration in backend
// backend/server.js
const cors = require('cors');

app.use(cors({
  origin: process.env.FRONTEND_URL || 'http://localhost:3000',
  credentials: true
}));
```

**Problem**: ML service not responding

```bash
# Check if service is running
curl http://localhost:8000/health

# Check logs
tail -f ml-service/logs/app.log

# Restart with verbose logging
uvicorn main:app --reload --log-level debug
```

### Authentication Issues

**Problem**: Token expired errors

```javascript
// Implement token refresh logic
// frontend/src/utils/api.js
import axios from 'axios';

axios.interceptors.response.use(
  response => response,
  async error => {
    if (error.response?.status === 401) {
      localStorage.removeItem('token');
      window.location.href = '/login';
    }
    return Promise.reject(error);
  }
);
```

### MongoDB Connection

**Problem**: Database connection fails

```javascript
// backend/config/database.js
const mongoose = require('mongoose');

const connectDB = async () => {
  try {
    await mongoose.connect(process.env.MONGODB_URI, {
      useNewUrlParser: true,
      useUnifiedTopology: true,
      serverSelectionTimeoutMS: 5000
    });
    console.log('MongoDB connected successfully');
  } catch (error) {
    console.error('MongoDB connection error:', error);
    process.exit(1);
  }
};

module.exports = connectDB;
```

### ML Model Issues

**Problem**: Model predictions returning errors

```python
# Add error handling and fallback
# ml-service/models/risk_predictor.py

def predict_risk(self, features):
    try:
        X = np.array(features).reshape(1, -1)
        
        # Validate input
        if len(features) != 5:
            raise ValueError(f"Expected 5 features, got {len(features)}")
        
        # Check for invalid values
        if any(f < 0 for f in features):
            raise ValueError("Features cannot be negative")
        
        # Predict
        risk_score = self.model.predict_proba(X)[0][1]
        
    except Exception as e:
        # Fallback to rule-based
        print(f"Prediction error: {e}, using fallback")
        risk_score = self._rule_based_risk(features)
    
    return self._format_response(risk_score, features)
```

## Common Usage Patterns

### Admin User Creation Flow

```javascript
// Complete flow: Create user → Assign tasks → Monitor analytics

const createUserWithTasks = async (userData, tasks) => {
  // 1. Create user
  const userResponse = await axios.post(`${API_URL}/api/users`, userData);
  const userId = userResponse.data.id;
  
  // 2. Assign tasks
  const taskPromises = tasks.map(task => 
    axios.post(`${API_URL}/api/tasks`, {
      ...task,
      assignedTo: userId
    })
  );
  await Promise.all(taskPromises);
  
  // 3. Get initial analytics
  const analytics = await axios.post(`${ML_API_URL}/api/ml/predict-risk`, {
    userId,
    loginFrequency: 0,
    taskCompletionRate: 0,
    avgResponseTime: 0,
    ticketsRaised: 0,
    failedLoginAttempts: 0
  });
  
  return { user: userResponse.data, analytics: analytics.data };
};
```

### Real-time Task Tracking

```javascript
// Track task time with stopwatch functionality

let taskTimer = null;
let elapsedSeconds = 0;

const startTaskTimer = (taskId) => {
  elapsedSeconds = 0;
  taskTimer = setInterval(() => {
    elapsedSeconds++;
    updateTimerDisplay(elapsedSeconds);
  }, 1000);
};

const stopTaskTimer = async (taskId) => {
  if (taskTimer) {
    clearInterval(taskTimer);
    
    // Save time to backend
    await axios.post(`${API_URL}/api/tasks/${taskId}/time`, {
      timeSpent: elapsedSeconds
    });
    
    taskTimer = null;
    elapsedSeconds = 0;
  }
};

const updateTimerDisplay = (seconds) => {
  const hours = Math.floor(seconds / 3600);
  const minutes = Math.floor((seconds % 3600) / 60);
  const secs = seconds % 60;
  
  document.getElementById('timer').textContent = 
    `${hours}:${minutes.toString().padStart(2, '0')}:${secs.toString().padStart(2, '0')}`;
};
```

This skill provides comprehensive coverage of the Enterprise User Management System with AI Analytics, enabling AI coding agents to effectively assist developers in implementing, configuring, and troubleshooting the system.
