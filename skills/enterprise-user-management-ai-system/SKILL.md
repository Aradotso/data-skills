---
name: enterprise-user-management-ai-system
description: Full-stack user management system with AI-powered analytics for risk detection, burnout analysis, and ticket classification
triggers:
  - "set up enterprise user management system"
  - "integrate AI analytics for user management"
  - "implement role-based access control with AI"
  - "create user dashboard with task tracking"
  - "add AI-powered ticket classification"
  - "build kanban board with user management"
  - "configure burnout detection for employees"
  - "deploy user management system with ML service"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## What This Project Does

The Enterprise User Management System is a full-stack application that combines traditional user and task management with AI-powered analytics. It provides:

- **User Management**: JWT-authenticated CRUD operations for users with role-based access control
- **Task Management**: Kanban board with time tracking and progress monitoring
- **Support Tickets**: AI-classified ticket routing and management
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, and project delay prediction
- **Admin Dashboard**: Organization-wide analytics and audit logs

The system consists of three main components:
1. **Frontend** (React.js) - User interface and dashboards
2. **Backend** (Node.js) - REST API and business logic
3. **ML Service** (FastAPI + scikit-learn) - AI/ML predictions and analytics

## Installation

### Prerequisites

```bash
# Required
node >= 14.x
npm >= 6.x
python >= 3.8
pip >= 21.x
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

Create `.env` file in backend directory:

```env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
JWT_SECRET=your_jwt_secret_key
JWT_EXPIRES_IN=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
```

Start backend:

```bash
npm start
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt
```

Create `.env` file in ml-service directory:

```env
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
MODEL_PATH=./models
LOG_LEVEL=info
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

Create `.env` file in frontend directory:

```env
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
```

Start frontend:

```bash
npm start
```

## Key API Endpoints

### Authentication

```javascript
// Login
POST /api/auth/login
{
  "email": "user@example.com",
  "password": "password123"
}

// Register
POST /api/auth/register
{
  "name": "John Doe",
  "email": "john@example.com",
  "password": "password123",
  "role": "user"
}

// Response includes JWT token
{
  "token": "eyJhbGciOiJIUzI1NiIs...",
  "user": { "id": "...", "name": "...", "role": "..." }
}
```

### User Management (Admin)

```javascript
// Get all users
GET /api/users
Headers: { "Authorization": "Bearer <token>" }

// Create user
POST /api/users
{
  "name": "Jane Smith",
  "email": "jane@example.com",
  "role": "user",
  "department": "Engineering"
}

// Update user
PUT /api/users/:userId
{
  "name": "Jane Smith Updated",
  "role": "admin"
}

// Delete user
DELETE /api/users/:userId
```

### Task Management

```javascript
// Get user tasks
GET /api/tasks/user/:userId
Headers: { "Authorization": "Bearer <token>" }

// Create task
POST /api/tasks
{
  "title": "Implement feature X",
  "description": "Add new functionality",
  "assignedTo": "userId",
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
  "duration": 3600, // seconds
  "notes": "Completed initial implementation"
}
```

### Support Tickets

```javascript
// Create ticket
POST /api/tickets
{
  "title": "Login issue",
  "description": "Cannot access dashboard after password reset",
  "priority": "high",
  "category": "technical"
}

// AI Classification (automatic on creation)
// Returns: { "category": "authentication", "priority": "high", "assignedTeam": "IT Support" }

// Update ticket
PATCH /api/tickets/:ticketId
{
  "status": "in-progress",
  "assignedTo": "supportUserId"
}
```

### AI Analytics Endpoints

```javascript
// Risk Prediction
POST /api/ai/risk-prediction
{
  "userId": "user123",
  "behaviorMetrics": {
    "loginFrequency": 15,
    "failedLogins": 3,
    "taskCompletionRate": 0.65,
    "averageResponseTime": 120
  }
}
// Response: { "riskScore": 0.72, "riskLevel": "high", "factors": [...] }

// Burnout Detection
POST /api/ai/burnout-detection
{
  "userId": "user123",
  "workloadMetrics": {
    "tasksAssigned": 25,
    "tasksCompleted": 15,
    "averageWorkHours": 52,
    "overtimeHours": 12,
    "ticketsHandled": 40
  }
}
// Response: { "burnoutScore": 0.85, "risk": "critical", "recommendations": [...] }

// Anomaly Detection
POST /api/ai/anomaly-detection
{
  "userId": "user123",
  "activityLog": [
    { "timestamp": "2026-04-15T10:00:00Z", "action": "login", "ip": "192.168.1.1" },
    { "timestamp": "2026-04-15T10:05:00Z", "action": "data_access", "resource": "user_database" }
  ]
}
// Response: { "isAnomaly": true, "anomalyScore": 0.89, "alerts": [...] }

// Project Delay Prediction
POST /api/ai/project-prediction
{
  "projectId": "proj123",
  "metrics": {
    "tasksTotal": 50,
    "tasksCompleted": 20,
    "daysElapsed": 30,
    "daysRemaining": 20,
    "teamSize": 5,
    "averageVelocity": 0.67
  }
}
// Response: { "delayProbability": 0.78, "estimatedDelay": 15, "recommendations": [...] }
```

## Frontend Usage Patterns

### Authentication Hook

```javascript
// src/hooks/useAuth.js
import { useState, useEffect } from 'react';
import axios from 'axios';

const useAuth = () => {
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
      const response = await axios.get(`${process.env.REACT_APP_API_URL}/api/auth/me`);
      setUser(response.data);
    } catch (error) {
      localStorage.removeItem('token');
    } finally {
      setLoading(false);
    }
  };

  const login = async (email, password) => {
    const response = await axios.post(`${process.env.REACT_APP_API_URL}/api/auth/login`, {
      email,
      password
    });
    localStorage.setItem('token', response.data.token);
    axios.defaults.headers.common['Authorization'] = `Bearer ${response.data.token}`;
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

export default useAuth;
```

### Kanban Board Component

```javascript
// src/components/KanbanBoard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const KanbanBoard = ({ userId }) => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });

  useEffect(() => {
    fetchTasks();
  }, [userId]);

  const fetchTasks = async () => {
    try {
      const response = await axios.get(`${process.env.REACT_APP_API_URL}/api/tasks/user/${userId}`);
      const categorized = {
        todo: response.data.filter(t => t.status === 'todo'),
        inProgress: response.data.filter(t => t.status === 'in-progress'),
        done: response.data.filter(t => t.status === 'done')
      };
      setTasks(categorized);
    } catch (error) {
      console.error('Failed to fetch tasks:', error);
    }
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    try {
      await axios.patch(`${process.env.REACT_APP_API_URL}/api/tasks/${taskId}/status`, {
        status: newStatus
      });
      fetchTasks(); // Refresh
    } catch (error) {
      console.error('Failed to update task:', error);
    }
  };

  return (
    <div className="kanban-board">
      <Column 
        title="To Do" 
        tasks={tasks.todo} 
        onStatusChange={(taskId) => updateTaskStatus(taskId, 'in-progress')}
      />
      <Column 
        title="In Progress" 
        tasks={tasks.inProgress} 
        onStatusChange={(taskId) => updateTaskStatus(taskId, 'done')}
      />
      <Column 
        title="Done" 
        tasks={tasks.done} 
      />
    </div>
  );
};

const Column = ({ title, tasks, onStatusChange }) => (
  <div className="kanban-column">
    <h3>{title}</h3>
    {tasks.map(task => (
      <div key={task.id} className="task-card" onClick={() => onStatusChange?.(task.id)}>
        <h4>{task.title}</h4>
        <p>{task.description}</p>
        <span className={`priority-${task.priority}`}>{task.priority}</span>
      </div>
    ))}
  </div>
);

export default KanbanBoard;
```

### AI Analytics Dashboard

```javascript
// src/components/AIAnalyticsDashboard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const AIAnalyticsDashboard = ({ userId }) => {
  const [analytics, setAnalytics] = useState({
    riskScore: null,
    burnoutScore: null,
    anomalies: []
  });

  useEffect(() => {
    fetchAnalytics();
  }, [userId]);

  const fetchAnalytics = async () => {
    try {
      const [risk, burnout, anomalies] = await Promise.all([
        axios.post(`${process.env.REACT_APP_ML_API_URL}/api/ai/risk-prediction`, { userId }),
        axios.post(`${process.env.REACT_APP_ML_API_URL}/api/ai/burnout-detection`, { userId }),
        axios.post(`${process.env.REACT_APP_ML_API_URL}/api/ai/anomaly-detection`, { userId })
      ]);

      setAnalytics({
        riskScore: risk.data.riskScore,
        burnoutScore: burnout.data.burnoutScore,
        anomalies: anomalies.data.alerts || []
      });
    } catch (error) {
      console.error('Failed to fetch analytics:', error);
    }
  };

  return (
    <div className="ai-analytics-dashboard">
      <div className="metric-card">
        <h3>Risk Score</h3>
        <div className={`score ${getRiskLevel(analytics.riskScore)}`}>
          {(analytics.riskScore * 100).toFixed(0)}%
        </div>
      </div>
      <div className="metric-card">
        <h3>Burnout Risk</h3>
        <div className={`score ${getBurnoutLevel(analytics.burnoutScore)}`}>
          {(analytics.burnoutScore * 100).toFixed(0)}%
        </div>
      </div>
      <div className="alerts-section">
        <h3>Security Alerts</h3>
        {analytics.anomalies.map((alert, idx) => (
          <div key={idx} className="alert-item">
            {alert.message}
          </div>
        ))}
      </div>
    </div>
  );
};

const getRiskLevel = (score) => {
  if (score > 0.7) return 'high';
  if (score > 0.4) return 'medium';
  return 'low';
};

const getBurnoutLevel = (score) => {
  if (score > 0.8) return 'critical';
  if (score > 0.5) return 'warning';
  return 'normal';
};

export default AIAnalyticsDashboard;
```

## Backend Implementation Patterns

### User Controller

```javascript
// backend/controllers/userController.js
const User = require('../models/User');
const jwt = require('jsonwebtoken');

exports.getAllUsers = async (req, res) => {
  try {
    const users = await User.find().select('-password');
    res.json(users);
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
};

exports.createUser = async (req, res) => {
  try {
    const { name, email, password, role, department } = req.body;
    
    const existingUser = await User.findOne({ email });
    if (existingUser) {
      return res.status(400).json({ message: 'User already exists' });
    }

    const user = new User({ name, email, password, role, department });
    await user.save();

    res.status(201).json({ 
      message: 'User created successfully',
      user: { id: user._id, name: user.name, email: user.email, role: user.role }
    });
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
};

exports.updateUser = async (req, res) => {
  try {
    const { userId } = req.params;
    const updates = req.body;

    const user = await User.findByIdAndUpdate(userId, updates, { new: true }).select('-password');
    if (!user) {
      return res.status(404).json({ message: 'User not found' });
    }

    res.json({ message: 'User updated successfully', user });
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
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
    res.status(500).json({ message: 'Server error', error: error.message });
  }
};
```

### Authentication Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

exports.authenticate = async (req, res, next) => {
  try {
    const token = req.headers.authorization?.replace('Bearer ', '');
    
    if (!token) {
      return res.status(401).json({ message: 'No token provided' });
    }

    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    const user = await User.findById(decoded.userId).select('-password');

    if (!user) {
      return res.status(401).json({ message: 'User not found' });
    }

    req.user = user;
    next();
  } catch (error) {
    res.status(401).json({ message: 'Invalid token', error: error.message });
  }
};

exports.authorizeAdmin = (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ message: 'Access denied. Admin only.' });
  }
  next();
};
```

### Task Model

```javascript
// backend/models/Task.js
const mongoose = require('mongoose');

const taskSchema = new mongoose.Schema({
  title: { type: String, required: true },
  description: { type: String },
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
  createdBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
  status: { 
    type: String, 
    enum: ['todo', 'in-progress', 'done'], 
    default: 'todo' 
  },
  priority: { 
    type: String, 
    enum: ['low', 'medium', 'high', 'critical'], 
    default: 'medium' 
  },
  dueDate: { type: Date },
  timeTracked: { type: Number, default: 0 }, // seconds
  createdAt: { type: Date, default: Date.now },
  updatedAt: { type: Date, default: Date.now }
});

taskSchema.pre('save', function(next) {
  this.updatedAt = Date.now();
  next();
});

module.exports = mongoose.model('Task', taskSchema);
```

## ML Service Implementation

### FastAPI Main Application

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from typing import List, Dict, Optional
import numpy as np
from sklearn.ensemble import IsolationForest, RandomForestClassifier
from river import anomaly
import pickle
import os

app = FastAPI(title="Enterprise User Management AI Service")

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Models storage
models = {}

# Load or initialize models
def load_models():
    model_path = os.getenv('MODEL_PATH', './models')
    if not os.path.exists(model_path):
        os.makedirs(model_path)
    
    # Initialize anomaly detection model
    models['anomaly'] = anomaly.HalfSpaceTrees(seed=42)
    
    # Initialize risk prediction model (placeholder - train with real data)
    models['risk'] = RandomForestClassifier(n_estimators=100, random_state=42)

load_models()

class RiskPredictionRequest(BaseModel):
    userId: str
    behaviorMetrics: Dict[str, float]

class BurnoutDetectionRequest(BaseModel):
    userId: str
    workloadMetrics: Dict[str, float]

class AnomalyDetectionRequest(BaseModel):
    userId: str
    activityLog: List[Dict]

class ProjectPredictionRequest(BaseModel):
    projectId: str
    metrics: Dict[str, float]

@app.post("/api/ai/risk-prediction")
async def predict_risk(request: RiskPredictionRequest):
    try:
        metrics = request.behaviorMetrics
        
        # Calculate risk score based on metrics
        risk_factors = []
        
        # Failed logins factor
        if metrics.get('failedLogins', 0) > 5:
            risk_factors.append(0.3)
        
        # Low task completion rate
        if metrics.get('taskCompletionRate', 1.0) < 0.7:
            risk_factors.append(0.25)
        
        # Unusual login frequency
        if metrics.get('loginFrequency', 0) < 5 or metrics.get('loginFrequency', 0) > 50:
            risk_factors.append(0.2)
        
        # High response time
        if metrics.get('averageResponseTime', 0) > 300:
            risk_factors.append(0.15)
        
        risk_score = sum(risk_factors) if risk_factors else 0.1
        risk_score = min(risk_score, 1.0)
        
        risk_level = 'high' if risk_score > 0.7 else 'medium' if risk_score > 0.4 else 'low'
        
        return {
            "riskScore": risk_score,
            "riskLevel": risk_level,
            "factors": risk_factors,
            "recommendations": generate_risk_recommendations(risk_score)
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ai/burnout-detection")
async def detect_burnout(request: BurnoutDetectionRequest):
    try:
        metrics = request.workloadMetrics
        
        # Calculate burnout score
        burnout_score = 0.0
        
        # High task load
        task_ratio = metrics.get('tasksCompleted', 0) / max(metrics.get('tasksAssigned', 1), 1)
        if task_ratio < 0.6:
            burnout_score += 0.25
        
        # Excessive work hours
        if metrics.get('averageWorkHours', 40) > 50:
            burnout_score += 0.3
        
        # Overtime
        if metrics.get('overtimeHours', 0) > 10:
            burnout_score += 0.2
        
        # High ticket volume
        if metrics.get('ticketsHandled', 0) > 30:
            burnout_score += 0.15
        
        burnout_score = min(burnout_score, 1.0)
        
        risk = 'critical' if burnout_score > 0.8 else 'warning' if burnout_score > 0.5 else 'normal'
        
        return {
            "burnoutScore": burnout_score,
            "risk": risk,
            "recommendations": generate_burnout_recommendations(burnout_score),
            "metrics": metrics
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ai/anomaly-detection")
async def detect_anomaly(request: AnomalyDetectionRequest):
    try:
        activity_log = request.activityLog
        
        # Simple anomaly detection based on patterns
        anomalies = []
        anomaly_score = 0.0
        
        # Check for unusual access patterns
        access_times = [log.get('timestamp') for log in activity_log]
        actions = [log.get('action') for log in activity_log]
        
        # Multiple failed logins
        failed_logins = sum(1 for action in actions if 'failed' in action.lower())
        if failed_logins > 3:
            anomalies.append("Multiple failed login attempts detected")
            anomaly_score += 0.4
        
        # Data access outside normal hours (simplified)
        sensitive_actions = sum(1 for action in actions if 'data_access' in action.lower())
        if sensitive_actions > 10:
            anomalies.append("Unusual data access pattern")
            anomaly_score += 0.3
        
        # Different IP addresses
        ips = set(log.get('ip') for log in activity_log if log.get('ip'))
        if len(ips) > 3:
            anomalies.append("Multiple IP addresses detected")
            anomaly_score += 0.2
        
        is_anomaly = anomaly_score > 0.5
        
        return {
            "isAnomaly": is_anomaly,
            "anomalyScore": min(anomaly_score, 1.0),
            "alerts": anomalies,
            "details": {
                "activitiesAnalyzed": len(activity_log),
                "failedLogins": failed_logins,
                "uniqueIPs": len(ips)
            }
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ai/project-prediction")
async def predict_project_delay(request: ProjectPredictionRequest):
    try:
        metrics = request.metrics
        
        # Calculate completion rate
        completion_rate = metrics.get('tasksCompleted', 0) / max(metrics.get('tasksTotal', 1), 1)
        time_elapsed_rate = metrics.get('daysElapsed', 0) / max(
            metrics.get('daysElapsed', 0) + metrics.get('daysRemaining', 1), 1
        )
        
        # Predict delay
        delay_probability = 0.0
        
        if completion_rate < time_elapsed_rate:
            delay_probability = min((time_elapsed_rate - completion_rate) * 2, 1.0)
        
        # Factor in velocity
        velocity = metrics.get('averageVelocity', 1.0)
        if velocity < 0.7:
            delay_probability += 0.2
        
        delay_probability = min(delay_probability, 1.0)
        
        # Estimate delay in days
        remaining_tasks = metrics.get('tasksTotal', 0) - metrics.get('tasksCompleted', 0)
        team_size = metrics.get('teamSize', 1)
        estimated_delay = int(remaining_tasks / max(team_size * velocity, 0.1)) if delay_probability > 0.5 else 0
        
        return {
            "delayProbability": delay_probability,
            "estimatedDelay": estimated_delay,
            "completionRate": completion_rate,
            "recommendations": generate_project_recommendations(delay_probability),
            "metrics": metrics
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

def generate_risk_recommendations(score: float) -> List[str]:
    if score > 0.7:
        return [
            "Enable two-factor authentication",
            "Review recent activity logs",
            "Reset password immediately",
            "Contact security team"
        ]
    elif score > 0.4:
        return [
            "Monitor user activity closely",
            "Review access permissions",
            "Schedule security training"
        ]
    return ["Continue normal monitoring"]

def generate_burnout_recommendations(score: float) -> List[str]:
    if score > 0.8:
        return [
            "Redistribute workload immediately",
            "Schedule mandatory time off",
            "Reduce overtime hours",
            "Provide mental health support"
        ]
    elif score > 0.5:
        return [
            "Review task assignments",
            "Monitor work hours",
            "Consider additional resources"
        ]
    return ["Workload appears normal"]

def generate_project_recommendations(probability: float) -> List[str]:
    if probability > 0.7:
        return [
            "Increase team size",
            "Reduce scope or extend deadline",
            "Address blockers immediately",
            "Daily standup meetings"
        ]
    elif probability > 0.4:
        return [
            "Monitor progress closely",
            "Identify potential risks",
            "Optimize workflow"
        ]
    return ["Project on track"]

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "ML Service"}
```

## Configuration

### Backend Routes Setup

```javascript
// backend/routes/index.js
const express = require('express');
const router = express.Router();
const { authenticate, authorizeAdmin } = require('../middleware/auth');

// Controllers
const authController = require('../controllers/authController');
const userController = require('../controllers/userController');
const taskController = require('../controllers/taskController');
const ticketController = require('../controllers/ticketController');

// Auth routes
router.post('/auth/register', authController.register);
router.post('/auth/login', authController.login);
router.get('/auth/me', authenticate, authController.getCurrentUser);

// User routes (admin only)
router.get('/users', authenticate, authorizeAdmin, userController.getAllUsers);
router.post('/users', authenticate, authorizeAdmin, userController.createUser);
router.put('/users/:userId', authenticate, authorizeAdmin, userController.updateUser);
router.delete('/users/:userId', authenticate, authorizeAdmin, userController.deleteUser);

// Task routes
router.get('/tasks/user/:userId', authenticate, taskController.getUserTasks);
router.post('/tasks', authenticate, taskController.createTask);
router.patch('/tasks/:taskId/status', authenticate, taskController.updateTaskStatus);
router.post('/tasks/:taskId/time', authenticate, taskController.trackTime);

// Ticket routes
router.get('/tickets', authenticate, ticketController.getAllTickets);
router.post('/tickets', authenticate, ticketController.createTicket);
router.patch('/tickets/:ticketId', authenticate, ticketController.updateTicket);

module.exports = router;
```

### Database Connection

```javascript
// backend/config/database.js
const mongoose = require('mongoose');

const connectDB = async () => {
  try {
    await mongoose.connect(process.env.MONGODB_URI, {
      useNewUrlParser: true,
      useUnifiedTopology: true,
    });
    console.log('MongoDB connected successfully');
  } catch (error) {
    console.error('MongoDB connection error:', error);
    process.exit(1);
  }
};

module.exports = connectDB;
```

## Common Patterns

### Time Tracking Component

```javascript
// src/components/TimeTracker.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const TimeTracker = ({ taskId }) => {
  const [isRunning, setIsRunning] = useState(false);
