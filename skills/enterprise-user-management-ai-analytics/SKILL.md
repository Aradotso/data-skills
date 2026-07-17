---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for risk detection, burnout analysis, and intelligent ticket routing
triggers:
  - "setup enterprise user management system"
  - "integrate AI analytics for user management"
  - "implement role-based access control with JWT"
  - "create user dashboard with task tracking"
  - "build ticket management system with AI classification"
  - "add anomaly detection to user system"
  - "configure ML service for user analytics"
  - "develop kanban board for task management"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## What This Project Does

Enterprise User Management System is a full-stack application that combines traditional user/task management with AI-driven analytics. It provides:

- **User Management**: Role-based access control, authentication via JWT
- **Task Management**: Kanban boards, time tracking, task assignment
- **Support System**: Intelligent ticket classification and routing
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, project delay prediction
- **Real-time Insights**: Dashboards for admins and users with performance metrics

The system is split into three services: React frontend, Node.js/Spring Boot backend, and FastAPI ML service.

## Installation

### Prerequisites

- Node.js 14+ and npm
- Python 3.8+
- MongoDB (local or cloud instance)

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
# Backend runs at http://localhost:5000
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt
```

Create `ml-service/.env`:

```env
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
MODEL_PATH=./models
LOG_LEVEL=INFO
```

Start ML service:

```bash
uvicorn main:app --reload --host 0.0.0.0 --port 8000
# ML service runs at http://localhost:8000
```

### Frontend Setup

```bash
cd frontend
npm install
```

Create `frontend/.env`:

```env
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_URL=http://localhost:8000
```

Start frontend:

```bash
npm start
# Frontend runs at http://localhost:3000
```

## Key API Endpoints

### Authentication (Backend)

```javascript
// POST /api/auth/register
{
  "name": "John Doe",
  "email": "john@company.com",
  "password": "securepass123",
  "role": "user"
}

// POST /api/auth/login
{
  "email": "john@company.com",
  "password": "securepass123"
}
// Returns: { token: "jwt_token", user: {...} }
```

### User Management (Backend)

```javascript
// GET /api/users - Get all users (Admin only)
// GET /api/users/:id - Get user by ID
// PUT /api/users/:id - Update user
// DELETE /api/users/:id - Delete user (Admin only)

// Example: Update user
fetch('http://localhost:5000/api/users/123', {
  method: 'PUT',
  headers: {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${token}`
  },
  body: JSON.stringify({
    name: "Jane Doe",
    role: "admin"
  })
});
```

### Task Management (Backend)

```javascript
// POST /api/tasks - Create task
{
  "title": "Implement user authentication",
  "description": "Add JWT-based auth",
  "assignedTo": "user_id",
  "status": "todo",
  "priority": "high",
  "dueDate": "2026-05-01"
}

// GET /api/tasks - Get all tasks
// GET /api/tasks/user/:userId - Get user's tasks
// PUT /api/tasks/:id - Update task status
// DELETE /api/tasks/:id - Delete task
```

### Support Tickets (Backend)

```javascript
// POST /api/tickets - Create ticket
{
  "subject": "Cannot access dashboard",
  "description": "Getting 403 error when trying to access admin panel",
  "priority": "high",
  "category": "technical"
}

// GET /api/tickets - Get all tickets
// PUT /api/tickets/:id - Update ticket
// POST /api/tickets/:id/classify - AI classify ticket
```

### AI Analytics (ML Service)

```python
# POST /api/ml/risk-prediction
{
  "userId": "user123",
  "loginAttempts": 5,
  "failedLogins": 3,
  "lastLoginTime": "2026-04-15T10:30:00Z",
  "accessPatterns": ["admin", "users", "logs"]
}
# Returns: { "riskScore": 0.75, "riskLevel": "high", "factors": [...] }

# POST /api/ml/anomaly-detection
{
  "userId": "user123",
  "behavior": {
    "loginHour": 3,
    "loginLocation": "Unknown",
    "actionsPerSession": 150
  }
}
# Returns: { "isAnomaly": true, "anomalyScore": 0.85, "details": "..." }

# POST /api/ml/burnout-analysis
{
  "userId": "user123",
  "tasksCompleted": 45,
  "averageWorkHours": 12,
  "weeklyTasks": 60,
  "overtimeHours": 20
}
# Returns: { "burnoutRisk": "high", "score": 0.82, "recommendations": [...] }

# POST /api/ml/project-insights
{
  "projectId": "proj123",
  "completedTasks": 30,
  "totalTasks": 100,
  "daysElapsed": 45,
  "teamSize": 5
}
# Returns: { "delayPrediction": "likely", "estimatedDelay": 15, "suggestions": [...] }
```

## Code Examples

### Frontend: User Authentication

```javascript
// src/services/authService.js
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL;

export const login = async (email, password) => {
  try {
    const response = await axios.post(`${API_URL}/auth/login`, {
      email,
      password
    });
    
    if (response.data.token) {
      localStorage.setItem('token', response.data.token);
      localStorage.setItem('user', JSON.stringify(response.data.user));
    }
    
    return response.data;
  } catch (error) {
    throw error.response?.data?.message || 'Login failed';
  }
};

export const logout = () => {
  localStorage.removeItem('token');
  localStorage.removeItem('user');
};

export const getCurrentUser = () => {
  return JSON.parse(localStorage.getItem('user'));
};

export const getAuthHeader = () => {
  const token = localStorage.getItem('token');
  return { Authorization: `Bearer ${token}` };
};
```

### Frontend: Task Management Component

```javascript
// src/components/KanbanBoard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import { getAuthHeader } from '../services/authService';

const KanbanBoard = ({ userId }) => {
  const [tasks, setTasks] = useState({
    todo: [],
    inProgress: [],
    done: []
  });

  useEffect(() => {
    fetchTasks();
  }, [userId]);

  const fetchTasks = async () => {
    try {
      const response = await axios.get(
        `${process.env.REACT_APP_API_URL}/tasks/user/${userId}`,
        { headers: getAuthHeader() }
      );
      
      const grouped = response.data.reduce((acc, task) => {
        acc[task.status] = acc[task.status] || [];
        acc[task.status].push(task);
        return acc;
      }, { todo: [], inProgress: [], done: [] });
      
      setTasks(grouped);
    } catch (error) {
      console.error('Failed to fetch tasks:', error);
    }
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    try {
      await axios.put(
        `${process.env.REACT_APP_API_URL}/tasks/${taskId}`,
        { status: newStatus },
        { headers: getAuthHeader() }
      );
      fetchTasks();
    } catch (error) {
      console.error('Failed to update task:', error);
    }
  };

  return (
    <div className="kanban-board">
      {['todo', 'inProgress', 'done'].map(status => (
        <div key={status} className="kanban-column">
          <h3>{status.toUpperCase()}</h3>
          {tasks[status].map(task => (
            <div key={task._id} className="task-card">
              <h4>{task.title}</h4>
              <p>{task.description}</p>
              <select 
                value={status}
                onChange={(e) => updateTaskStatus(task._id, e.target.value)}
              >
                <option value="todo">To Do</option>
                <option value="inProgress">In Progress</option>
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

### Backend: JWT Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

const protect = async (req, res, next) => {
  let token;

  if (req.headers.authorization && req.headers.authorization.startsWith('Bearer')) {
    try {
      token = req.headers.authorization.split(' ')[1];
      const decoded = jwt.verify(token, process.env.JWT_SECRET);
      
      req.user = await User.findById(decoded.id).select('-password');
      
      if (!req.user) {
        return res.status(401).json({ message: 'User not found' });
      }
      
      next();
    } catch (error) {
      return res.status(401).json({ message: 'Not authorized, token failed' });
    }
  }

  if (!token) {
    return res.status(401).json({ message: 'Not authorized, no token' });
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

### Backend: User Controller

```javascript
// backend/controllers/userController.js
const User = require('../models/User');
const bcrypt = require('bcryptjs');
const jwt = require('jsonwebtoken');

// Register user
exports.registerUser = async (req, res) => {
  try {
    const { name, email, password, role } = req.body;

    const userExists = await User.findOne({ email });
    if (userExists) {
      return res.status(400).json({ message: 'User already exists' });
    }

    const hashedPassword = await bcrypt.hash(password, 10);

    const user = await User.create({
      name,
      email,
      password: hashedPassword,
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
    res.status(500).json({ message: error.message });
  }
};

// Get all users (Admin only)
exports.getAllUsers = async (req, res) => {
  try {
    const users = await User.find().select('-password');
    res.json(users);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
};

// Update user
exports.updateUser = async (req, res) => {
  try {
    const { id } = req.params;
    const updates = req.body;

    if (updates.password) {
      updates.password = await bcrypt.hash(updates.password, 10);
    }

    const user = await User.findByIdAndUpdate(id, updates, { new: true })
      .select('-password');

    if (!user) {
      return res.status(404).json({ message: 'User not found' });
    }

    res.json(user);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
};
```

### ML Service: Risk Prediction Model

```python
# ml-service/models/risk_predictor.py
from sklearn.ensemble import RandomForestClassifier
import numpy as np
import pickle
import os

class RiskPredictor:
    def __init__(self, model_path='./models/risk_model.pkl'):
        self.model_path = model_path
        self.model = None
        self.load_or_create_model()
    
    def load_or_create_model(self):
        if os.path.exists(self.model_path):
            with open(self.model_path, 'rb') as f:
                self.model = pickle.load(f)
        else:
            self.model = RandomForestClassifier(n_estimators=100, random_state=42)
            # Train with initial dummy data
            X_train = np.random.rand(100, 5)
            y_train = np.random.randint(0, 2, 100)
            self.model.fit(X_train, y_train)
            self.save_model()
    
    def save_model(self):
        os.makedirs(os.path.dirname(self.model_path), exist_ok=True)
        with open(self.model_path, 'wb') as f:
            pickle.dump(self.model, f)
    
    def predict_risk(self, features):
        """
        features: dict with keys:
        - loginAttempts: int
        - failedLogins: int
        - timeSinceLastLogin: float (hours)
        - uniqueAccessPatterns: int
        - offHoursAccess: int (0 or 1)
        """
        X = np.array([[
            features.get('loginAttempts', 0),
            features.get('failedLogins', 0),
            features.get('timeSinceLastLogin', 0),
            features.get('uniqueAccessPatterns', 0),
            features.get('offHoursAccess', 0)
        ]])
        
        risk_prob = self.model.predict_proba(X)[0][1]
        
        if risk_prob > 0.7:
            risk_level = "high"
        elif risk_prob > 0.4:
            risk_level = "medium"
        else:
            risk_level = "low"
        
        factors = []
        if features.get('failedLogins', 0) > 3:
            factors.append("Multiple failed login attempts")
        if features.get('offHoursAccess', 0) == 1:
            factors.append("Off-hours access detected")
        if features.get('uniqueAccessPatterns', 0) > 10:
            factors.append("Unusual access patterns")
        
        return {
            "riskScore": float(risk_prob),
            "riskLevel": risk_level,
            "factors": factors
        }
```

### ML Service: FastAPI Endpoints

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from models.risk_predictor import RiskPredictor
from models.anomaly_detector import AnomalyDetector
from models.burnout_analyzer import BurnoutAnalyzer
from typing import List, Optional
import os
from dotenv import load_dotenv

load_dotenv()

app = FastAPI(title="Enterprise User Management ML Service")

# Initialize models
risk_predictor = RiskPredictor()
anomaly_detector = AnomalyDetector()
burnout_analyzer = BurnoutAnalyzer()

class RiskRequest(BaseModel):
    userId: str
    loginAttempts: int
    failedLogins: int
    timeSinceLastLogin: float
    uniqueAccessPatterns: int
    offHoursAccess: int

class AnomalyRequest(BaseModel):
    userId: str
    loginHour: int
    actionsPerSession: int
    dataAccessed: int
    locationChange: int

class BurnoutRequest(BaseModel):
    userId: str
    tasksCompleted: int
    averageWorkHours: float
    weeklyTasks: int
    overtimeHours: float

@app.post("/api/ml/risk-prediction")
async def predict_risk(request: RiskRequest):
    try:
        features = {
            'loginAttempts': request.loginAttempts,
            'failedLogins': request.failedLogins,
            'timeSinceLastLogin': request.timeSinceLastLogin,
            'uniqueAccessPatterns': request.uniqueAccessPatterns,
            'offHoursAccess': request.offHoursAccess
        }
        
        result = risk_predictor.predict_risk(features)
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/anomaly-detection")
async def detect_anomaly(request: AnomalyRequest):
    try:
        features = {
            'loginHour': request.loginHour,
            'actionsPerSession': request.actionsPerSession,
            'dataAccessed': request.dataAccessed,
            'locationChange': request.locationChange
        }
        
        result = anomaly_detector.detect(features)
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/burnout-analysis")
async def analyze_burnout(request: BurnoutRequest):
    try:
        features = {
            'tasksCompleted': request.tasksCompleted,
            'averageWorkHours': request.averageWorkHours,
            'weeklyTasks': request.weeklyTasks,
            'overtimeHours': request.overtimeHours
        }
        
        result = burnout_analyzer.analyze(features)
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "ml-analytics"}

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

### ML Service: Burnout Analyzer

```python
# ml-service/models/burnout_analyzer.py
import numpy as np
from sklearn.preprocessing import StandardScaler

class BurnoutAnalyzer:
    def __init__(self):
        self.scaler = StandardScaler()
        # Threshold values
        self.max_safe_hours = 8
        self.max_safe_tasks = 40
        self.max_safe_overtime = 5
    
    def analyze(self, features):
        """
        Analyze burnout risk based on work metrics
        """
        work_hours = features.get('averageWorkHours', 8)
        weekly_tasks = features.get('weeklyTasks', 20)
        overtime = features.get('overtimeHours', 0)
        tasks_completed = features.get('tasksCompleted', 10)
        
        # Calculate burnout score (0-1)
        hours_factor = min(work_hours / self.max_safe_hours, 2.0) - 1.0
        tasks_factor = min(weekly_tasks / self.max_safe_tasks, 2.0) - 1.0
        overtime_factor = min(overtime / self.max_safe_overtime, 2.0) - 1.0
        
        # Weighted average
        burnout_score = (
            hours_factor * 0.4 +
            tasks_factor * 0.3 +
            overtime_factor * 0.3
        )
        burnout_score = max(0.0, min(1.0, burnout_score))
        
        if burnout_score > 0.7:
            risk_level = "high"
            recommendations = [
                "Reduce workload immediately",
                "Schedule time off",
                "Redistribute tasks to team members",
                "Consult with HR about workload"
            ]
        elif burnout_score > 0.4:
            risk_level = "medium"
            recommendations = [
                "Monitor work hours closely",
                "Consider task prioritization",
                "Take regular breaks",
                "Discuss workload with manager"
            ]
        else:
            risk_level = "low"
            recommendations = [
                "Maintain current pace",
                "Continue healthy work-life balance"
            ]
        
        return {
            "burnoutRisk": risk_level,
            "score": float(burnout_score),
            "recommendations": recommendations,
            "metrics": {
                "averageWorkHours": work_hours,
                "weeklyTasks": weekly_tasks,
                "overtimeHours": overtime
            }
        }
```

## Common Patterns

### Protecting Routes with JWT

```javascript
// backend/routes/taskRoutes.js
const express = require('express');
const router = express.Router();
const { protect, adminOnly } = require('../middleware/auth');
const taskController = require('../controllers/taskController');

// User routes (protected)
router.get('/user/:userId', protect, taskController.getUserTasks);
router.post('/', protect, taskController.createTask);
router.put('/:id', protect, taskController.updateTask);

// Admin routes (protected + admin only)
router.get('/', protect, adminOnly, taskController.getAllTasks);
router.delete('/:id', protect, adminOnly, taskController.deleteTask);

module.exports = router;
```

### Integrating ML Service in Backend

```javascript
// backend/services/mlService.js
const axios = require('axios');

const ML_SERVICE_URL = process.env.ML_SERVICE_URL || 'http://localhost:8000';

exports.predictRisk = async (userData) => {
  try {
    const response = await axios.post(`${ML_SERVICE_URL}/api/ml/risk-prediction`, {
      userId: userData.userId,
      loginAttempts: userData.loginAttempts,
      failedLogins: userData.failedLogins,
      timeSinceLastLogin: userData.timeSinceLastLogin,
      uniqueAccessPatterns: userData.uniqueAccessPatterns,
      offHoursAccess: userData.offHoursAccess
    });
    return response.data;
  } catch (error) {
    console.error('ML Service Error:', error.message);
    throw new Error('Risk prediction failed');
  }
};

exports.analyzeBurnout = async (userMetrics) => {
  try {
    const response = await axios.post(`${ML_SERVICE_URL}/api/ml/burnout-analysis`, {
      userId: userMetrics.userId,
      tasksCompleted: userMetrics.tasksCompleted,
      averageWorkHours: userMetrics.averageWorkHours,
      weeklyTasks: userMetrics.weeklyTasks,
      overtimeHours: userMetrics.overtimeHours
    });
    return response.data;
  } catch (error) {
    console.error('ML Service Error:', error.message);
    throw new Error('Burnout analysis failed');
  }
};
```

### Frontend: AI Analytics Dashboard

```javascript
// src/components/AdminAnalytics.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import { getAuthHeader } from '../services/authService';

const AdminAnalytics = () => {
  const [analytics, setAnalytics] = useState({
    highRiskUsers: [],
    burnoutAlerts: [],
    anomalies: []
  });

  useEffect(() => {
    fetchAnalytics();
  }, []);

  const fetchAnalytics = async () => {
    try {
      const response = await axios.get(
        `${process.env.REACT_APP_API_URL}/analytics/dashboard`,
        { headers: getAuthHeader() }
      );
      setAnalytics(response.data);
    } catch (error) {
      console.error('Failed to fetch analytics:', error);
    }
  };

  const analyzeUserRisk = async (userId) => {
    try {
      const response = await axios.post(
        `${process.env.REACT_APP_API_URL}/analytics/risk/${userId}`,
        {},
        { headers: getAuthHeader() }
      );
      alert(`Risk Level: ${response.data.riskLevel}\nScore: ${response.data.riskScore}`);
    } catch (error) {
      console.error('Risk analysis failed:', error);
    }
  };

  return (
    <div className="analytics-dashboard">
      <h2>AI Analytics Dashboard</h2>
      
      <div className="analytics-section">
        <h3>High Risk Users</h3>
        {analytics.highRiskUsers.map(user => (
          <div key={user.id} className="alert-card risk-alert">
            <p><strong>{user.name}</strong></p>
            <p>Risk Score: {user.riskScore.toFixed(2)}</p>
            <p>Factors: {user.factors.join(', ')}</p>
            <button onClick={() => analyzeUserRisk(user.id)}>
              Re-analyze
            </button>
          </div>
        ))}
      </div>

      <div className="analytics-section">
        <h3>Burnout Alerts</h3>
        {analytics.burnoutAlerts.map(alert => (
          <div key={alert.userId} className="alert-card burnout-alert">
            <p><strong>{alert.userName}</strong></p>
            <p>Burnout Risk: {alert.riskLevel}</p>
            <p>Avg Hours: {alert.avgHours}h/day</p>
            <ul>
              {alert.recommendations.map((rec, idx) => (
                <li key={idx}>{rec}</li>
              ))}
            </ul>
          </div>
        ))}
      </div>
    </div>
  );
};

export default AdminAnalytics;
```

## Configuration

### MongoDB Schema Examples

```javascript
// backend/models/User.js
const mongoose = require('mongoose');

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
  avatar: String,
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

module.exports = mongoose.model('User', userSchema);
```

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
  assignedBy: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User'
  },
  status: {
    type: String,
    enum: ['todo', 'inProgress', 'done'],
    default: 'todo'
  },
  priority: {
    type: String,
    enum: ['low', 'medium', 'high', 'urgent'],
    default: 'medium'
  },
  dueDate: Date,
  timeTracked: {
    type: Number,
    default: 0 // in minutes
  },
  createdAt: {
    type: Date,
    default: Date.now
  },
  completedAt: Date
});

module.exports = mongoose.model('Task', taskSchema);
```

### Environment Variables Reference

**Backend (.env)**:
```env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
JWT_SECRET=your_very_secure_secret_key_here
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
CORS_ORIGIN=http://localhost:3000
```

**ML Service (.env)**:
```env
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
MODEL_PATH=./models
LOG_LEVEL=INFO
```

**Frontend (.env)**:
```env
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_URL=http://localhost:8000
```

## Troubleshooting

### JWT Token Issues

**Problem**: "Not authorized, token failed" error

**Solution**:
```javascript
// Ensure token is properly stored and sent
const token = localStorage.getItem('token');
if (!token) {
  // Redirect to login
  window.location.href = '/login';
}

// Check token expiration
const decoded = jwt.decode(token);
if (decoded.exp * 1000 < Date.now()) {
  // Token expired, refresh or re-login
  localStorage.removeItem('token');
  window.location.href = '/login';
}
```

### MongoDB Connection Errors

**Problem**: Cannot connect to MongoDB

**Solution**:
```javascript
// backend/config/db.js
const mongoose = require('mongoose');

const connectDB = async () => {
  try {
    const conn = await mongoose.connect(process.env.MONGODB_URI, {
      useNewUrlParser: true,
      useUnifiedTopology: true,
      serverSelectionTimeoutMS: 5000
    });
    console.log(`MongoDB Connected: ${conn.connection.host}`);
  } catch (error) {
    console.error(`Error
