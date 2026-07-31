---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics, risk detection, and task management built with React, Node.js, and FastAPI ML service
triggers:
  - how do I set up the enterprise user management system
  - implement AI analytics for user management
  - create a user management dashboard with AI insights
  - integrate ML service for risk and anomaly detection
  - build a task tracking system with burnout analysis
  - set up JWT authentication for user management
  - configure AI-powered ticket classification system
  - deploy user management system with AI features
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A comprehensive full-stack enterprise user management platform that combines traditional CRUD operations with AI-powered analytics including risk detection, anomaly detection, burnout analysis, and predictive insights. Built with React frontend, Node.js/Express backend, MongoDB database, and FastAPI ML service.

## What This Project Does

This system provides:
- **User Management**: Role-based access control with admin/user permissions
- **Task Management**: Kanban-style board with time tracking and progress monitoring
- **Support Tickets**: AI-classified ticket system with smart routing
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, and project delay prediction
- **Authentication**: JWT-based secure authentication
- **Dashboards**: Separate admin and user dashboards with real-time insights

## Installation

### Prerequisites
```bash
# Required
node >= 14.x
python >= 3.8
mongodb >= 4.x
npm or yarn
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
MONGODB_URI=mongodb://localhost:27017/enterprise_user_management
JWT_SECRET=your_jwt_secret_key_here
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
EOF

# Start backend
npm start
```

### ML Service Setup
```bash
cd ml-service
pip install -r requirements.txt

# Create .env file
cat > .env << EOF
API_HOST=0.0.0.0
API_PORT=8000
MODEL_PATH=./models
LOG_LEVEL=INFO
EOF

# Start ML service
uvicorn main:app --reload --host 0.0.0.0 --port 8000
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
```

## Architecture Overview

```
┌─────────────┐         ┌─────────────┐         ┌─────────────┐
│   React     │────────▶│   Node.js   │────────▶│   MongoDB   │
│  Frontend   │         │   Backend   │         │   Database  │
└─────────────┘         └─────────────┘         └─────────────┘
                              │
                              ▼
                        ┌─────────────┐
                        │   FastAPI   │
                        │  ML Service │
                        └─────────────┘
```

## Key API Endpoints

### Authentication
```javascript
// POST /api/auth/register
{
  "username": "john_doe",
  "email": "john@example.com",
  "password": "securePassword123",
  "role": "user"
}

// POST /api/auth/login
{
  "email": "john@example.com",
  "password": "securePassword123"
}
// Returns: { token: "jwt_token", user: {...} }
```

### User Management (Admin)
```javascript
// GET /api/users - Get all users
// GET /api/users/:id - Get user by ID
// PUT /api/users/:id - Update user
// DELETE /api/users/:id - Delete user

// Example: Update user role
fetch('http://localhost:5000/api/users/123', {
  method: 'PUT',
  headers: {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${token}`
  },
  body: JSON.stringify({
    role: 'admin',
    status: 'active'
  })
})
```

### Task Management
```javascript
// POST /api/tasks - Create task
{
  "title": "Implement AI Analytics",
  "description": "Add burnout detection feature",
  "assignedTo": "user_id",
  "priority": "high",
  "status": "todo",
  "dueDate": "2026-05-01"
}

// PUT /api/tasks/:id - Update task status
{
  "status": "in-progress",
  "timeSpent": 3600
}

// GET /api/tasks/user/:userId - Get user tasks
```

### Support Tickets
```javascript
// POST /api/tickets - Create ticket
{
  "title": "Cannot access dashboard",
  "description": "Getting 403 error when logging in",
  "priority": "high",
  "category": "technical"
}

// AI Classification happens automatically via ML service
```

### ML/AI Endpoints

```python
# POST /api/ml/risk-prediction
{
  "user_id": "123",
  "login_attempts": 5,
  "failed_logins": 2,
  "session_duration": 7200,
  "last_activity": "2026-04-15T10:30:00Z"
}
# Returns: { "risk_score": 0.75, "risk_level": "high" }

# POST /api/ml/anomaly-detection
{
  "user_id": "123",
  "activity_data": {
    "login_time": "03:00",
    "ip_address": "192.168.1.100",
    "location": "Unknown",
    "device": "Unknown"
  }
}
# Returns: { "is_anomaly": true, "confidence": 0.89 }

# POST /api/ml/burnout-analysis
{
  "user_id": "123",
  "tasks_completed": 45,
  "tasks_overdue": 12,
  "avg_work_hours": 11.5,
  "stress_indicators": ["late_nights", "weekend_work"]
}
# Returns: { "burnout_risk": 0.82, "recommendation": "reduce_workload" }

# POST /api/ml/project-prediction
{
  "project_id": "proj_456",
  "tasks_total": 100,
  "tasks_completed": 45,
  "days_elapsed": 30,
  "days_remaining": 20,
  "team_velocity": 1.2
}
# Returns: { "completion_probability": 0.65, "estimated_delay_days": 5 }
```

## Frontend Implementation

### Authentication Hook
```javascript
// src/hooks/useAuth.js
import { useState, useEffect } from 'react';

export const useAuth = () => {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    const token = localStorage.getItem('token');
    if (token) {
      // Validate token and fetch user
      fetchUser(token);
    }
    setLoading(false);
  }, []);

  const login = async (email, password) => {
    const response = await fetch(`${process.env.REACT_APP_API_URL}/api/auth/login`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ email, password })
    });
    
    const data = await response.json();
    if (data.token) {
      localStorage.setItem('token', data.token);
      setUser(data.user);
    }
    return data;
  };

  const logout = () => {
    localStorage.removeItem('token');
    setUser(null);
  };

  return { user, loading, login, logout };
};
```

### Task Dashboard Component
```javascript
// src/components/TaskDashboard.jsx
import React, { useState, useEffect } from 'react';

const TaskDashboard = () => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });
  const token = localStorage.getItem('token');

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    const response = await fetch(`${process.env.REACT_APP_API_URL}/api/tasks/user/me`, {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    const data = await response.json();
    
    setTasks({
      todo: data.filter(t => t.status === 'todo'),
      inProgress: data.filter(t => t.status === 'in-progress'),
      done: data.filter(t => t.status === 'done')
    });
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    await fetch(`${process.env.REACT_APP_API_URL}/api/tasks/${taskId}`, {
      method: 'PUT',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${token}`
      },
      body: JSON.stringify({ status: newStatus })
    });
    fetchTasks();
  };

  return (
    <div className="kanban-board">
      <div className="column">
        <h3>To Do ({tasks.todo.length})</h3>
        {tasks.todo.map(task => (
          <TaskCard key={task._id} task={task} onMove={updateTaskStatus} />
        ))}
      </div>
      <div className="column">
        <h3>In Progress ({tasks.inProgress.length})</h3>
        {tasks.inProgress.map(task => (
          <TaskCard key={task._id} task={task} onMove={updateTaskStatus} />
        ))}
      </div>
      <div className="column">
        <h3>Done ({tasks.done.length})</h3>
        {tasks.done.map(task => (
          <TaskCard key={task._id} task={task} onMove={updateTaskStatus} />
        ))}
      </div>
    </div>
  );
};
```

### AI Analytics Integration
```javascript
// src/services/aiService.js
const ML_API_URL = process.env.REACT_APP_ML_API_URL;

export const getRiskPrediction = async (userId) => {
  const response = await fetch(`${ML_API_URL}/api/ml/risk-prediction`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ user_id: userId })
  });
  return response.json();
};

export const checkAnomaly = async (activityData) => {
  const response = await fetch(`${ML_API_URL}/api/ml/anomaly-detection`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(activityData)
  });
  return response.json();
};

export const analyzeBurnout = async (userData) => {
  const response = await fetch(`${ML_API_URL}/api/ml/burnout-analysis`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(userData)
  });
  return response.json();
};

// Usage in component
import { analyzeBurnout } from '../services/aiService';

const BurnoutWidget = ({ userId }) => {
  const [burnoutData, setBurnoutData] = useState(null);

  useEffect(() => {
    const fetchBurnoutAnalysis = async () => {
      const data = await analyzeBurnout({ user_id: userId });
      setBurnoutData(data);
    };
    fetchBurnoutAnalysis();
  }, [userId]);

  return (
    <div className="burnout-widget">
      {burnoutData && (
        <div className={`risk-level-${burnoutData.risk_level}`}>
          <h4>Burnout Risk: {(burnoutData.burnout_risk * 100).toFixed(0)}%</h4>
          <p>{burnoutData.recommendation}</p>
        </div>
      )}
    </div>
  );
};
```

## Backend Implementation

### User Model (MongoDB/Mongoose)
```javascript
// backend/models/User.js
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');

const userSchema = new mongoose.Schema({
  username: {
    type: String,
    required: true,
    unique: true
  },
  email: {
    type: String,
    required: true,
    unique: true
  },
  password: {
    type: String,
    required: true
  },
  role: {
    type: String,
    enum: ['admin', 'user', 'manager'],
    default: 'user'
  },
  status: {
    type: String,
    enum: ['active', 'inactive', 'suspended'],
    default: 'active'
  },
  profileImage: String,
  department: String,
  lastLogin: Date,
  loginAttempts: {
    type: Number,
    default: 0
  }
}, { timestamps: true });

// Hash password before saving
userSchema.pre('save', async function(next) {
  if (!this.isModified('password')) return next();
  this.password = await bcrypt.hash(this.password, 10);
  next();
});

// Method to compare passwords
userSchema.methods.comparePassword = async function(candidatePassword) {
  return bcrypt.compare(candidatePassword, this.password);
};

module.exports = mongoose.model('User', userSchema);
```

### Task Model
```javascript
// backend/models/Task.js
const mongoose = require('mongoose');

const taskSchema = new mongoose.Schema({
  title: {
    type: String,
    required: true
  },
  description: String,
  assignedTo: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User',
    required: true
  },
  createdBy: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User'
  },
  status: {
    type: String,
    enum: ['todo', 'in-progress', 'done'],
    default: 'todo'
  },
  priority: {
    type: String,
    enum: ['low', 'medium', 'high', 'urgent'],
    default: 'medium'
  },
  dueDate: Date,
  timeSpent: {
    type: Number,
    default: 0
  },
  tags: [String],
  attachments: [String]
}, { timestamps: true });

module.exports = mongoose.model('Task', taskSchema);
```

### Authentication Middleware
```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

const authMiddleware = async (req, res, next) => {
  try {
    const token = req.header('Authorization')?.replace('Bearer ', '');
    
    if (!token) {
      return res.status(401).json({ message: 'Authentication required' });
    }

    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    const user = await User.findById(decoded.userId).select('-password');
    
    if (!user || user.status !== 'active') {
      return res.status(401).json({ message: 'Invalid authentication' });
    }

    req.user = user;
    next();
  } catch (error) {
    res.status(401).json({ message: 'Invalid token' });
  }
};

const adminMiddleware = (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ message: 'Admin access required' });
  }
  next();
};

module.exports = { authMiddleware, adminMiddleware };
```

### Task Routes
```javascript
// backend/routes/tasks.js
const express = require('express');
const router = express.Router();
const Task = require('../models/Task');
const { authMiddleware, adminMiddleware } = require('../middleware/auth');

// Get user's tasks
router.get('/user/me', authMiddleware, async (req, res) => {
  try {
    const tasks = await Task.find({ assignedTo: req.user._id })
      .populate('createdBy', 'username email')
      .sort({ createdAt: -1 });
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Create task (admin only)
router.post('/', authMiddleware, adminMiddleware, async (req, res) => {
  try {
    const task = new Task({
      ...req.body,
      createdBy: req.user._id
    });
    await task.save();
    res.status(201).json(task);
  } catch (error) {
    res.status(400).json({ message: error.message });
  }
});

// Update task
router.put('/:id', authMiddleware, async (req, res) => {
  try {
    const task = await Task.findById(req.params.id);
    
    if (!task) {
      return res.status(404).json({ message: 'Task not found' });
    }

    // Users can only update their own tasks
    if (task.assignedTo.toString() !== req.user._id.toString() && req.user.role !== 'admin') {
      return res.status(403).json({ message: 'Access denied' });
    }

    Object.assign(task, req.body);
    await task.save();
    res.json(task);
  } catch (error) {
    res.status(400).json({ message: error.message });
  }
});

// Track time
router.post('/:id/time', authMiddleware, async (req, res) => {
  try {
    const { timeSpent } = req.body;
    const task = await Task.findByIdAndUpdate(
      req.params.id,
      { $inc: { timeSpent: timeSpent } },
      { new: true }
    );
    res.json(task);
  } catch (error) {
    res.status(400).json({ message: error.message });
  }
});

module.exports = router;
```

## ML Service Implementation

### FastAPI ML Service Structure
```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
import numpy as np
from typing import List, Optional
import pickle
import os

app = FastAPI(title="Enterprise User Management ML Service")

# CORS configuration
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Load or initialize models
MODEL_PATH = os.getenv("MODEL_PATH", "./models")

# Risk prediction model
class RiskPredictionRequest(BaseModel):
    user_id: str
    login_attempts: int
    failed_logins: int
    session_duration: float
    last_activity: str
    unusual_activity_count: Optional[int] = 0

class RiskPredictionResponse(BaseModel):
    risk_score: float
    risk_level: str
    factors: List[str]

@app.post("/api/ml/risk-prediction", response_model=RiskPredictionResponse)
async def predict_risk(request: RiskPredictionRequest):
    try:
        # Calculate risk score based on features
        risk_score = 0.0
        factors = []
        
        # Failed login attempts
        if request.failed_logins > 3:
            risk_score += 0.3
            factors.append("high_failed_logins")
        
        # Login attempts
        if request.login_attempts > 10:
            risk_score += 0.2
            factors.append("excessive_login_attempts")
        
        # Session duration anomaly
        if request.session_duration > 28800:  # 8 hours
            risk_score += 0.15
            factors.append("extended_session")
        
        # Unusual activity
        if request.unusual_activity_count > 5:
            risk_score += 0.35
            factors.append("unusual_activity_detected")
        
        # Determine risk level
        if risk_score >= 0.7:
            risk_level = "high"
        elif risk_score >= 0.4:
            risk_level = "medium"
        else:
            risk_level = "low"
        
        return RiskPredictionResponse(
            risk_score=min(risk_score, 1.0),
            risk_level=risk_level,
            factors=factors
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

# Anomaly detection
class AnomalyDetectionRequest(BaseModel):
    user_id: str
    activity_data: dict

class AnomalyDetectionResponse(BaseModel):
    is_anomaly: bool
    confidence: float
    anomaly_type: Optional[str]

@app.post("/api/ml/anomaly-detection", response_model=AnomalyDetectionResponse)
async def detect_anomaly(request: AnomalyDetectionRequest):
    try:
        is_anomaly = False
        confidence = 0.0
        anomaly_type = None
        
        activity = request.activity_data
        
        # Check for unusual login time
        if 'login_time' in activity:
            hour = int(activity['login_time'].split(':')[0])
            if hour < 6 or hour > 22:
                is_anomaly = True
                confidence = 0.75
                anomaly_type = "unusual_login_time"
        
        # Check for unknown location
        if activity.get('location') == 'Unknown':
            is_anomaly = True
            confidence = max(confidence, 0.65)
            anomaly_type = "unknown_location"
        
        # Check for new device
        if activity.get('device') == 'Unknown':
            is_anomaly = True
            confidence = max(confidence, 0.60)
            anomaly_type = "new_device"
        
        return AnomalyDetectionResponse(
            is_anomaly=is_anomaly,
            confidence=confidence,
            anomaly_type=anomaly_type
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

# Burnout analysis
class BurnoutAnalysisRequest(BaseModel):
    user_id: str
    tasks_completed: int
    tasks_overdue: int
    avg_work_hours: float
    stress_indicators: List[str]

class BurnoutAnalysisResponse(BaseModel):
    burnout_risk: float
    risk_level: str
    recommendation: str

@app.post("/api/ml/burnout-analysis", response_model=BurnoutAnalysisResponse)
async def analyze_burnout(request: BurnoutAnalysisRequest):
    try:
        burnout_risk = 0.0
        
        # Overdue tasks impact
        if request.tasks_overdue > 0:
            overdue_ratio = request.tasks_overdue / max(request.tasks_completed, 1)
            burnout_risk += min(overdue_ratio * 0.4, 0.4)
        
        # Work hours impact
        if request.avg_work_hours > 9:
            burnout_risk += 0.3
        elif request.avg_work_hours > 10:
            burnout_risk += 0.5
        
        # Stress indicators
        stress_weight = len(request.stress_indicators) * 0.1
        burnout_risk += min(stress_weight, 0.3)
        
        burnout_risk = min(burnout_risk, 1.0)
        
        # Determine risk level and recommendation
        if burnout_risk >= 0.7:
            risk_level = "high"
            recommendation = "Immediate workload reduction recommended. Schedule time off."
        elif burnout_risk >= 0.4:
            risk_level = "medium"
            recommendation = "Monitor workload closely. Consider redistributing tasks."
        else:
            risk_level = "low"
            recommendation = "Maintain current work-life balance."
        
        return BurnoutAnalysisResponse(
            burnout_risk=burnout_risk,
            risk_level=risk_level,
            recommendation=recommendation
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

# Project prediction
class ProjectPredictionRequest(BaseModel):
    project_id: str
    tasks_total: int
    tasks_completed: int
    days_elapsed: int
    days_remaining: int
    team_velocity: float

class ProjectPredictionResponse(BaseModel):
    completion_probability: float
    estimated_delay_days: int
    recommendation: str

@app.post("/api/ml/project-prediction", response_model=ProjectPredictionResponse)
async def predict_project(request: ProjectPredictionRequest):
    try:
        # Calculate completion rate
        completion_rate = request.tasks_completed / request.tasks_total
        expected_rate = request.days_elapsed / (request.days_elapsed + request.days_remaining)
        
        # Calculate velocity factor
        tasks_per_day = request.tasks_completed / max(request.days_elapsed, 1)
        required_velocity = (request.tasks_total - request.tasks_completed) / max(request.days_remaining, 1)
        
        velocity_ratio = (tasks_per_day * request.team_velocity) / max(required_velocity, 0.1)
        
        # Calculate completion probability
        if velocity_ratio >= 1.2:
            completion_probability = 0.95
            estimated_delay = 0
            recommendation = "Project on track for early completion"
        elif velocity_ratio >= 0.9:
            completion_probability = 0.75
            estimated_delay = 0
            recommendation = "Project on track"
        elif velocity_ratio >= 0.7:
            completion_probability = 0.50
            estimated_delay = int(request.days_remaining * 0.2)
            recommendation = "Minor delays expected. Monitor closely."
        else:
            completion_probability = 0.30
            estimated_delay = int(request.days_remaining * 0.4)
            recommendation = "Significant delays expected. Increase resources or adjust timeline."
        
        return ProjectPredictionResponse(
            completion_probability=completion_probability,
            estimated_delay_days=estimated_delay,
            recommendation=recommendation
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "ML Analytics"}

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

### ML Service Requirements
```txt
# ml-service/requirements.txt
fastapi==0.104.1
uvicorn==0.24.0
pydantic==2.5.0
numpy==1.24.3
scikit-learn==1.3.2
river==0.19.0
python-dotenv==1.0.0
```

## Common Patterns

### Protected Routes Pattern
```javascript
// frontend/src/components/ProtectedRoute.jsx
import { Navigate } from 'react-router-dom';
import { useAuth } from '../hooks/useAuth';

const ProtectedRoute = ({ children, requireAdmin = false }) => {
  const { user, loading } = useAuth();

  if (loading) return <div>Loading...</div>;
  
  if (!user) return <Navigate to="/login" />;
  
  if (requireAdmin && user.role !== 'admin') {
    return <Navigate to="/dashboard" />;
  }

  return children;
};

// Usage in App.js
import { BrowserRouter, Routes, Route } from 'react-router-dom';

function App() {
  return (
    <BrowserRouter>
      <Routes>
        <Route path="/login" element={<Login />} />
        <Route path="/dashboard" element={
          <ProtectedRoute>
            <UserDashboard />
          </ProtectedRoute>
        } />
        <Route path="/admin" element={
          <ProtectedRoute requireAdmin={true}>
            <AdminDashboard />
          </ProtectedRoute>
        } />
      </Routes>
    </BrowserRouter>
  );
}
```

### Real-time Notifications Pattern
```javascript
// backend/utils/notifications.js
const sendNotification = async (userId, notification) => {
  const Notification = require('../models/Notification');
  
  const newNotification = new Notification({
    userId,
    title: notification.title,
    message: notification.message,
    type: notification.type,
    read: false
  });
  
  await newNotification.save();
  
  // If WebSocket is configured, emit real-time
  // io.to(userId).emit('notification', newNotification);
  
  return newNotification;
};

module.exports = { sendNotification };

// Usage in task creation
const { sendNotification } = require('../utils/notifications');

router.post('/', authMiddleware, adminMiddleware, async (req, res) => {
  const task = new Task({...req.body, createdBy: req.user._id});
  await task.save();
  
  // Notify assigned user
  await sendNotification(task.assignedTo, {
    title: 'New Task Assigned',
    message: `You have been assigned: ${task.title}`,
    type: 'task_assigned'
  });
  
  res.status(201).json(task);
});
```

### AI Analytics Dashboard Pattern
```javascript
// frontend/src/components/AdminAnalytics.jsx
import React, { useState, useEffect } from 'react';
import { getRiskPrediction, analyzeBurnout } from '../services/aiService';

const AdminAnalytics = () => {
  const [analytics, setAnalytics] = useState({
    highRiskUsers: [],
    burnoutRisk: [],
    anomalies: []
  });

  useEffect(() => {
    fetchAnalytics();
  }, []);

  const fetchAnalytics = async () => {
    const token = localStorage.getItem('token');
    
    // Fetch all users
    const usersResponse = await fetch(`${process.env.REACT_APP_API_URL}/api/users`, {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    const users = await usersResponse.json();
    
    // Analyze each user
    const highRisk = [];
    const burnout = [];
    
    for (const user of users) {
      const riskData = await getRiskPrediction(user._id);
      if (riskData.risk_level === 'high') {
        highRisk.push({ ...user, riskData });
      
