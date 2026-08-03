---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics, task management, and predictive insights
triggers:
  - "set up enterprise user management system"
  - "implement AI-powered user analytics"
  - "create task management with burnout detection"
  - "build user management dashboard with AI insights"
  - "integrate ML predictions for project delays"
  - "configure JWT authentication for enterprise app"
  - "deploy user management system with FastAPI ML service"
  - "implement kanban board with time tracking"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI coding agents to work with a full-stack enterprise user management system that combines React frontend, Node.js backend, and FastAPI ML service for AI-powered analytics including risk detection, anomaly detection, burnout analysis, and predictive insights.

## What This Project Does

Enterprise User Management System provides:
- **User Management**: Secure authentication, role-based access control (RBAC)
- **Task Management**: Kanban boards, time tracking, assignment workflows
- **Support Tickets**: AI-based classification and routing
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, project delay prediction
- **Admin Dashboard**: Organization analytics, audit logs, user monitoring

**Architecture**: React frontend → Node.js/Express backend → MongoDB → FastAPI ML service

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
```

Create `.env` file:
```env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
JWT_SECRET=your_jwt_secret_key
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
```

Start backend:
```bash
npm start
# Or for development with hot reload
npm run dev
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt
```

Create `ml-service/.env`:
```env
MODEL_PATH=./models
LOG_LEVEL=INFO
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
```

Start ML service:
```bash
uvicorn main:app --reload --host 0.0.0.0 --port 8000
```

### Frontend Setup

```bash
cd frontend
npm install
```

Create `frontend/.env`:
```env
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_SERVICE_URL=http://localhost:8000
```

Start frontend:
```bash
npm start
```

## Key API Endpoints

### Authentication (Backend)

```javascript
// Register new user
POST /api/auth/register
{
  "name": "John Doe",
  "email": "john@example.com",
  "password": "SecurePass123",
  "role": "user"
}

// Login
POST /api/auth/login
{
  "email": "john@example.com",
  "password": "SecurePass123"
}
// Returns: { token, user }

// Get current user
GET /api/auth/me
Headers: { Authorization: "Bearer <token>" }
```

### User Management (Backend)

```javascript
// Get all users (Admin only)
GET /api/users
Headers: { Authorization: "Bearer <admin_token>" }

// Update user
PUT /api/users/:userId
Headers: { Authorization: "Bearer <token>" }
{
  "name": "Updated Name",
  "role": "admin"
}

// Delete user (Admin only)
DELETE /api/users/:userId
Headers: { Authorization: "Bearer <admin_token>" }
```

### Task Management (Backend)

```javascript
// Create task
POST /api/tasks
Headers: { Authorization: "Bearer <token>" }
{
  "title": "Implement feature X",
  "description": "Add new analytics dashboard",
  "assignedTo": "user_id",
  "priority": "high",
  "dueDate": "2026-05-01",
  "estimatedHours": 8
}

// Update task status
PUT /api/tasks/:taskId
{
  "status": "in-progress",
  "actualHours": 3
}

// Get user tasks
GET /api/tasks/user/:userId
```

### Support Tickets (Backend)

```javascript
// Create ticket
POST /api/tickets
Headers: { Authorization: "Bearer <token>" }
{
  "subject": "Login issue",
  "description": "Cannot access dashboard after password reset",
  "priority": "medium"
}

// AI classify ticket (triggers ML service)
POST /api/tickets/:ticketId/classify
// Returns: { category, suggestedAssignee, priority }
```

### AI Analytics Endpoints (ML Service)

```python
# Risk prediction
POST http://localhost:8000/api/predict/risk
{
  "userId": "user_123",
  "loginFrequency": 15,
  "failedLoginAttempts": 2,
  "tasksCompleted": 45,
  "ticketsRaised": 3,
  "averageTaskTime": 4.5
}
# Returns: { riskScore: 0.23, riskLevel: "low" }

# Anomaly detection
POST http://localhost:8000/api/detect/anomaly
{
  "userId": "user_123",
  "activityLog": [
    {"timestamp": "2026-04-15T10:00:00Z", "action": "login"},
    {"timestamp": "2026-04-15T10:05:00Z", "action": "data_export"}
  ]
}
# Returns: { isAnomaly: true, confidence: 0.87 }

# Burnout prediction
POST http://localhost:8000/api/predict/burnout
{
  "userId": "user_123",
  "weeklyHours": 55,
  "taskCount": 12,
  "avgTaskDuration": 6.5,
  "overdueCount": 4
}
# Returns: { burnoutRisk: 0.72, recommendation: "reduce workload" }

# Project delay prediction
POST http://localhost:8000/api/predict/delay
{
  "projectId": "proj_456",
  "tasksTotal": 20,
  "tasksCompleted": 8,
  "daysRemaining": 15,
  "teamSize": 5,
  "avgVelocity": 2.3
}
# Returns: { delayProbability: 0.65, estimatedDelay: 5 }
```

## Frontend Usage Patterns

### Authentication Flow

```javascript
// frontend/src/services/authService.js
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL;

export const login = async (email, password) => {
  const response = await axios.post(`${API_URL}/auth/login`, {
    email,
    password
  });
  
  if (response.data.token) {
    localStorage.setItem('token', response.data.token);
    localStorage.setItem('user', JSON.stringify(response.data.user));
  }
  
  return response.data;
};

export const getAuthHeader = () => {
  const token = localStorage.getItem('token');
  return { Authorization: `Bearer ${token}` };
};

export const logout = () => {
  localStorage.removeItem('token');
  localStorage.removeItem('user');
};
```

### Fetching User Dashboard Data

```javascript
// frontend/src/components/UserDashboard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import { getAuthHeader } from '../services/authService';

const UserDashboard = () => {
  const [tasks, setTasks] = useState([]);
  const [analytics, setAnalytics] = useState(null);
  
  useEffect(() => {
    fetchUserData();
  }, []);
  
  const fetchUserData = async () => {
    try {
      const user = JSON.parse(localStorage.getItem('user'));
      const API_URL = process.env.REACT_APP_API_URL;
      
      // Fetch tasks
      const tasksResponse = await axios.get(
        `${API_URL}/tasks/user/${user._id}`,
        { headers: getAuthHeader() }
      );
      setTasks(tasksResponse.data);
      
      // Fetch AI analytics
      const analyticsResponse = await axios.post(
        `${process.env.REACT_APP_ML_SERVICE_URL}/api/predict/burnout`,
        {
          userId: user._id,
          weeklyHours: calculateWeeklyHours(tasksResponse.data),
          taskCount: tasksResponse.data.length,
          avgTaskDuration: calculateAvgDuration(tasksResponse.data),
          overdueCount: tasksResponse.data.filter(t => t.overdue).length
        }
      );
      setAnalytics(analyticsResponse.data);
    } catch (error) {
      console.error('Error fetching dashboard data:', error);
    }
  };
  
  return (
    <div className="dashboard">
      <h1>My Dashboard</h1>
      {analytics && (
        <div className="burnout-alert">
          Burnout Risk: {(analytics.burnoutRisk * 100).toFixed(1)}%
        </div>
      )}
      <TaskList tasks={tasks} />
    </div>
  );
};
```

### Kanban Board Implementation

```javascript
// frontend/src/components/KanbanBoard.jsx
import React, { useState } from 'react';
import axios from 'axios';
import { getAuthHeader } from '../services/authService';

const KanbanBoard = ({ tasks, onTaskUpdate }) => {
  const columns = ['todo', 'in-progress', 'done'];
  
  const handleDragStart = (e, taskId) => {
    e.dataTransfer.setData('taskId', taskId);
  };
  
  const handleDrop = async (e, newStatus) => {
    e.preventDefault();
    const taskId = e.dataTransfer.getData('taskId');
    
    try {
      await axios.put(
        `${process.env.REACT_APP_API_URL}/tasks/${taskId}`,
        { status: newStatus },
        { headers: getAuthHeader() }
      );
      onTaskUpdate();
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };
  
  const handleDragOver = (e) => {
    e.preventDefault();
  };
  
  return (
    <div className="kanban-board">
      {columns.map(status => (
        <div
          key={status}
          className="kanban-column"
          onDrop={(e) => handleDrop(e, status)}
          onDragOver={handleDragOver}
        >
          <h3>{status.toUpperCase()}</h3>
          {tasks.filter(t => t.status === status).map(task => (
            <div
              key={task._id}
              className="task-card"
              draggable
              onDragStart={(e) => handleDragStart(e, task._id)}
            >
              <h4>{task.title}</h4>
              <p>{task.description}</p>
              <span className={`priority-${task.priority}`}>
                {task.priority}
              </span>
            </div>
          ))}
        </div>
      ))}
    </div>
  );
};

export default KanbanBoard;
```

## Backend Implementation Patterns

### JWT Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

const protect = async (req, res, next) => {
  let token;
  
  if (req.headers.authorization?.startsWith('Bearer')) {
    token = req.headers.authorization.split(' ')[1];
  }
  
  if (!token) {
    return res.status(401).json({ error: 'Not authorized' });
  }
  
  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = await User.findById(decoded.id).select('-password');
    next();
  } catch (error) {
    return res.status(401).json({ error: 'Invalid token' });
  }
};

const authorize = (...roles) => {
  return (req, res, next) => {
    if (!roles.includes(req.user.role)) {
      return res.status(403).json({ 
        error: 'Insufficient permissions' 
      });
    }
    next();
  };
};

module.exports = { protect, authorize };
```

### User Controller

```javascript
// backend/controllers/userController.js
const User = require('../models/User');
const jwt = require('jsonwebtoken');

// Register user
exports.register = async (req, res) => {
  try {
    const { name, email, password, role } = req.body;
    
    const userExists = await User.findOne({ email });
    if (userExists) {
      return res.status(400).json({ error: 'User already exists' });
    }
    
    const user = await User.create({
      name,
      email,
      password, // Should be hashed in User model pre-save hook
      role: role || 'user'
    });
    
    const token = jwt.sign(
      { id: user._id },
      process.env.JWT_SECRET,
      { expiresIn: process.env.JWT_EXPIRE }
    );
    
    res.status(201).json({
      token,
      user: {
        id: user._id,
        name: user.name,
        email: user.email,
        role: user.role
      }
    });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
};

// Get all users (Admin)
exports.getUsers = async (req, res) => {
  try {
    const users = await User.find().select('-password');
    res.json(users);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
};

// Update user
exports.updateUser = async (req, res) => {
  try {
    const { userId } = req.params;
    const updates = req.body;
    
    // Only admin can change roles
    if (updates.role && req.user.role !== 'admin') {
      delete updates.role;
    }
    
    const user = await User.findByIdAndUpdate(
      userId,
      updates,
      { new: true, runValidators: true }
    ).select('-password');
    
    res.json(user);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
};
```

### Task Routes

```javascript
// backend/routes/taskRoutes.js
const express = require('express');
const router = express.Router();
const { protect, authorize } = require('../middleware/auth');
const {
  createTask,
  getTasks,
  updateTask,
  deleteTask,
  getUserTasks
} = require('../controllers/taskController');

router.post('/', protect, authorize('admin', 'manager'), createTask);
router.get('/', protect, getTasks);
router.get('/user/:userId', protect, getUserTasks);
router.put('/:taskId', protect, updateTask);
router.delete('/:taskId', protect, authorize('admin'), deleteTask);

module.exports = router;
```

## ML Service Implementation

### FastAPI Main Application

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
import joblib
import numpy as np
from typing import List, Dict
import os
from dotenv import load_dotenv

load_dotenv()

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

class RiskPredictionRequest(BaseModel):
    userId: str
    loginFrequency: int
    failedLoginAttempts: int
    tasksCompleted: int
    ticketsRaised: int
    averageTaskTime: float

class BurnoutPredictionRequest(BaseModel):
    userId: str
    weeklyHours: float
    taskCount: int
    avgTaskDuration: float
    overdueCount: int

class DelayPredictionRequest(BaseModel):
    projectId: str
    tasksTotal: int
    tasksCompleted: int
    daysRemaining: int
    teamSize: int
    avgVelocity: float

@app.post("/api/predict/risk")
async def predict_risk(request: RiskPredictionRequest):
    """Predict user risk score based on behavior patterns"""
    try:
        # Feature engineering
        features = np.array([[
            request.loginFrequency,
            request.failedLoginAttempts,
            request.tasksCompleted,
            request.ticketsRaised,
            request.averageTaskTime
        ]])
        
        # Simple risk calculation (replace with trained model)
        risk_score = (
            request.failedLoginAttempts * 0.3 +
            (request.ticketsRaised / max(request.tasksCompleted, 1)) * 0.4 +
            (request.averageTaskTime / 10) * 0.3
        ) / 3
        
        risk_level = "low" if risk_score < 0.3 else "medium" if risk_score < 0.7 else "high"
        
        return {
            "userId": request.userId,
            "riskScore": round(risk_score, 3),
            "riskLevel": risk_level,
            "factors": {
                "failedLogins": request.failedLoginAttempts,
                "taskEfficiency": round(request.tasksCompleted / max(request.ticketsRaised, 1), 2)
            }
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/predict/burnout")
async def predict_burnout(request: BurnoutPredictionRequest):
    """Predict employee burnout risk"""
    try:
        # Burnout calculation
        workload_factor = min(request.weeklyHours / 40, 2)
        overdue_factor = request.overdueCount / max(request.taskCount, 1)
        duration_factor = request.avgTaskDuration / 8
        
        burnout_risk = (
            workload_factor * 0.4 +
            overdue_factor * 0.3 +
            duration_factor * 0.3
        ) / 3
        
        burnout_risk = min(burnout_risk, 1.0)
        
        recommendation = "healthy workload"
        if burnout_risk > 0.7:
            recommendation = "critical - immediate intervention needed"
        elif burnout_risk > 0.5:
            recommendation = "reduce workload"
        elif burnout_risk > 0.3:
            recommendation = "monitor closely"
        
        return {
            "userId": request.userId,
            "burnoutRisk": round(burnout_risk, 3),
            "recommendation": recommendation,
            "metrics": {
                "weeklyHours": request.weeklyHours,
                "overduePercentage": round(overdue_factor * 100, 1)
            }
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/predict/delay")
async def predict_delay(request: DelayPredictionRequest):
    """Predict project delay probability"""
    try:
        completion_rate = request.tasksCompleted / max(request.tasksTotal, 1)
        required_velocity = (request.tasksTotal - request.tasksCompleted) / max(request.daysRemaining, 1)
        velocity_gap = (required_velocity - request.avgVelocity) / max(request.avgVelocity, 0.1)
        
        delay_probability = min(1.0, max(0.0, velocity_gap))
        estimated_delay = max(0, int(velocity_gap * request.daysRemaining))
        
        return {
            "projectId": request.projectId,
            "delayProbability": round(delay_probability, 3),
            "estimatedDelay": estimated_delay,
            "completionRate": round(completion_rate * 100, 1),
            "requiredVelocity": round(required_velocity, 2),
            "currentVelocity": request.avgVelocity
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "ml-analytics"}
```

### Anomaly Detection with River

```python
# ml-service/services/anomaly_detector.py
from river import anomaly
from river import preprocessing
from typing import Dict, List
import json

class AnomalyDetector:
    def __init__(self):
        self.model = preprocessing.StandardScaler() | anomaly.HalfSpaceTrees(
            n_trees=10,
            height=8,
            window_size=250,
            seed=42
        )
        
    def detect(self, user_id: str, activity_log: List[Dict]) -> Dict:
        """Detect anomalies in user activity"""
        features = self._extract_features(activity_log)
        
        # Score the activity
        score = self.model.score_one(features)
        
        # Update model with new data
        self.model.learn_one(features)
        
        # Threshold for anomaly (can be tuned)
        is_anomaly = score > 0.6
        
        return {
            "userId": user_id,
            "isAnomaly": is_anomaly,
            "confidence": round(score, 3),
            "suspiciousActivities": self._identify_suspicious(activity_log) if is_anomaly else []
        }
    
    def _extract_features(self, activity_log: List[Dict]) -> Dict:
        """Extract numerical features from activity log"""
        return {
            "activity_count": len(activity_log),
            "unique_actions": len(set(a["action"] for a in activity_log)),
            "data_exports": sum(1 for a in activity_log if "export" in a.get("action", "")),
            "failed_attempts": sum(1 for a in activity_log if "failed" in a.get("action", "")),
            "off_hours_activity": sum(1 for a in activity_log if self._is_off_hours(a.get("timestamp")))
        }
    
    def _is_off_hours(self, timestamp: str) -> bool:
        """Check if activity occurred outside business hours"""
        from datetime import datetime
        dt = datetime.fromisoformat(timestamp.replace("Z", "+00:00"))
        return dt.hour < 6 or dt.hour > 22
    
    def _identify_suspicious(self, activity_log: List[Dict]) -> List[str]:
        """Identify specific suspicious activities"""
        suspicious = []
        for activity in activity_log:
            if "export" in activity.get("action", "").lower():
                suspicious.append(f"Data export at {activity['timestamp']}")
            if "delete" in activity.get("action", "").lower():
                suspicious.append(f"Delete operation at {activity['timestamp']}")
        return suspicious
```

## Database Models

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
    enum: ['user', 'admin', 'manager'],
    default: 'user'
  },
  department: String,
  isActive: {
    type: Boolean,
    default: true
  },
  loginHistory: [{
    timestamp: Date,
    ipAddress: String,
    success: Boolean
  }],
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
  return await bcrypt.compare(candidatePassword, this.password);
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
  assignedTo: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User',
    required: true
  },
  createdBy: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User'
  },
  dueDate: Date,
  estimatedHours: Number,
  actualHours: Number,
  tags: [String],
  timeTracking: [{
    startTime: Date,
    endTime: Date,
    duration: Number
  }],
  createdAt: {
    type: Date,
    default: Date.now
  },
  updatedAt: Date
});

taskSchema.pre('save', function(next) {
  this.updatedAt = Date.now();
  next();
});

module.exports = mongoose.model('Task', taskSchema);
```

## Configuration

### Backend Environment Variables

```env
# Server
PORT=5000
NODE_ENV=development

# Database
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt

# JWT
JWT_SECRET=your_secure_random_secret_key
JWT_EXPIRE=7d

# ML Service
ML_SERVICE_URL=http://localhost:8000

# Email (optional)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASS=your_app_password

# Rate Limiting
RATE_LIMIT_WINDOW_MS=900000
RATE_LIMIT_MAX_REQUESTS=100
```

### Frontend Environment Variables

```env
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_SERVICE_URL=http://localhost:8000
REACT_APP_ENVIRONMENT=development
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
pymongo==4.6.0
joblib==1.3.2
```

## Common Workflows

### Creating a Complete User Workflow

```javascript
// backend/workflows/userWorkflow.js
const User = require('../models/User');
const axios = require('axios');

exports.createUserWithAnalytics = async (userData) => {
  // 1. Create user
  const user = await User.create(userData);
  
  // 2. Initialize AI profile
  await axios.post(`${process.env.ML_SERVICE_URL}/api/init/user`, {
    userId: user._id,
    role: user.role,
    department: user.department
  });
  
  // 3. Send welcome email (if configured)
  // await sendWelcomeEmail(user.email);
  
  // 4. Create audit log
  await AuditLog.create({
    action: 'USER_CREATED',
    userId: user._id,
    details: { email: user.email, role: user.role }
  });
  
  return user;
};
```

### AI-Powered Task Assignment

```javascript
// backend/services/taskAssignmentService.js
const axios = require('axios');
const Task = require('../models/Task');
const User = require('../models/User');

exports.intelligentTaskAssignment = async (taskData) => {
  // Get all available users
  const users = await User.find({ 
    isActive: true,
    role: { $in: ['user', 'manager'] }
  });
  
  // Get workload and burnout risk for each user
  const userAnalytics = await Promise.all(
    users.map(async (user) => {
      const tasks = await Task.find({ 
        assignedTo: user._id,
        status: { $ne: 'done' }
      });
      
      const burnoutData = await axios.post(
        `${process.env.ML_SERVICE_URL}/api/predict/burnout`,
        {
          userId: user._id.toString(),
          weeklyHours: calculateWeeklyHours(tasks),
          taskCount: tasks.length,
          avgTaskDuration: calculateAvgDuration(tasks),
          overdueCount: tasks.filter(t => isOverdue(t)).length
        }
      );
      
      return {
        userId: user._id,
        burnoutRisk: burnoutData.data.burnoutRisk,
        currentTasks: tasks.length
      };
    })
  );
  
  // Select user with lowest burnout risk
  const bestUser = userAnalytics.reduce((best, current) => 
    current.burnoutRisk < best.burnoutRisk ? current : best
  );
  
  // Create task with optimal assignment
  const task = await Task.create({
    ...taskData,
    assignedTo: bestUser.userId
  });
  
  return task;
};

function calculateWeeklyHours(tasks) {
  const weekAgo = new Date(Date.now() - 7 * 24 * 60 * 60 * 1000);
  return tasks
    .filter(t => t.createdAt > weekAgo)
    .reduce((sum, t) => sum + (t.actualHours || t.estimatedHours || 0), 0);
}
```

## Troubleshooting

### MongoDB Connection Issues

```javascript
// backend/config/database.js
const
