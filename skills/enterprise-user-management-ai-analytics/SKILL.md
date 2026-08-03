---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for risk detection, burnout analysis, and predictive insights
triggers:
  - "set up enterprise user management with AI analytics"
  - "integrate AI-powered user management system"
  - "implement risk detection and anomaly detection for users"
  - "build admin dashboard with task management"
  - "create user management system with ML insights"
  - "add AI analytics to user management app"
  - "deploy enterprise user management with FastAPI ML service"
  - "configure JWT authentication for user management"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI coding agents to help developers implement and extend an enterprise-grade user management system with integrated AI analytics. The system provides user/admin dashboards, task management with Kanban boards, support ticket systems, and ML-powered insights including risk prediction, anomaly detection, burnout analysis, and predictive project analytics.

## What This Project Does

The Enterprise User Management System is a three-tier application:
- **Frontend (React)**: User/admin dashboards, Kanban boards, time tracking, ticket management
- **Backend (Node.js/Express)**: REST APIs, JWT authentication, MongoDB integration
- **ML Service (FastAPI)**: AI-powered analytics using scikit-learn and River for online learning

Key capabilities:
- Role-based access control (users vs admins)
- Task assignment and tracking with Kanban workflow
- Support ticket management with AI classification
- Real-time anomaly detection and risk scoring
- Burnout prediction based on workload analysis
- Predictive project delay insights

## Installation

### Prerequisites
- Node.js 14+ and npm
- Python 3.8+
- MongoDB instance (local or cloud)

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
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
JWT_SECRET=${JWT_SECRET}
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
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
LOG_LEVEL=info
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

## Key API Endpoints

### Authentication
```javascript
// POST /api/auth/register
{
  "username": "john_doe",
  "email": "john@example.com",
  "password": "securePass123",
  "role": "user" // or "admin"
}

// POST /api/auth/login
{
  "email": "john@example.com",
  "password": "securePass123"
}
// Returns: { token: "jwt_token", user: {...} }
```

### User Management (Admin)
```javascript
// GET /api/users - List all users
// GET /api/users/:id - Get user details
// PUT /api/users/:id - Update user
// DELETE /api/users/:id - Delete user
```

### Task Management
```javascript
// GET /api/tasks - Get all tasks
// POST /api/tasks - Create task
{
  "title": "Implement authentication",
  "description": "Add JWT-based auth",
  "assignedTo": "user_id",
  "status": "todo", // todo, in_progress, done
  "priority": "high",
  "dueDate": "2026-05-01"
}

// PUT /api/tasks/:id - Update task status
// DELETE /api/tasks/:id - Delete task
```

### Support Tickets
```javascript
// GET /api/tickets - Get all tickets
// POST /api/tickets - Create ticket
{
  "subject": "Cannot access dashboard",
  "description": "Getting 403 error",
  "priority": "high",
  "category": "technical"
}

// PUT /api/tickets/:id - Update ticket
// POST /api/tickets/:id/classify - AI classify ticket
```

### AI Analytics Endpoints
```javascript
// POST /api/ml/predict-risk - User risk prediction
{
  "userId": "user_id",
  "features": {
    "loginAttempts": 5,
    "taskCompletionRate": 0.75,
    "avgResponseTime": 24
  }
}

// POST /api/ml/detect-anomaly - Anomaly detection
{
  "userId": "user_id",
  "behavior": {
    "loginTime": "03:00",
    "location": "unknown",
    "activityPattern": "unusual"
  }
}

// POST /api/ml/predict-burnout - Burnout analysis
{
  "userId": "user_id",
  "workload": {
    "tasksAssigned": 15,
    "avgWorkHours": 10,
    "overtimeHours": 20
  }
}

// POST /api/ml/predict-delay - Project delay prediction
{
  "projectId": "project_id",
  "metrics": {
    "completionRate": 0.6,
    "daysRemaining": 10,
    "teamSize": 5
  }
}
```

## Backend Code Examples

### User Authentication Middleware (Node.js)

```javascript
// middleware/auth.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

exports.protect = async (req, res, next) => {
  let token;
  
  if (req.headers.authorization && req.headers.authorization.startsWith('Bearer')) {
    token = req.headers.authorization.split(' ')[1];
  }
  
  if (!token) {
    return res.status(401).json({ message: 'Not authorized to access this route' });
  }
  
  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = await User.findById(decoded.id);
    next();
  } catch (err) {
    return res.status(401).json({ message: 'Not authorized to access this route' });
  }
};

exports.authorize = (...roles) => {
  return (req, res, next) => {
    if (!roles.includes(req.user.role)) {
      return res.status(403).json({ 
        message: `User role ${req.user.role} is not authorized to access this route` 
      });
    }
    next();
  };
};
```

### Task Controller

```javascript
// controllers/taskController.js
const Task = require('../models/Task');
const User = require('../models/User');

exports.getTasks = async (req, res) => {
  try {
    const query = req.user.role === 'admin' 
      ? {} 
      : { assignedTo: req.user.id };
    
    const tasks = await Task.find(query)
      .populate('assignedTo', 'username email')
      .populate('createdBy', 'username')
      .sort({ createdAt: -1 });
    
    res.status(200).json({ success: true, data: tasks });
  } catch (error) {
    res.status(500).json({ success: false, message: error.message });
  }
};

exports.createTask = async (req, res) => {
  try {
    const task = await Task.create({
      ...req.body,
      createdBy: req.user.id
    });
    
    // Optionally call ML service to predict completion time
    const mlResponse = await fetch(`${process.env.ML_SERVICE_URL}/predict-completion`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        taskComplexity: req.body.priority,
        assignedUserId: req.body.assignedTo
      })
    });
    const prediction = await mlResponse.json();
    
    res.status(201).json({ 
      success: true, 
      data: task,
      estimatedCompletion: prediction.days
    });
  } catch (error) {
    res.status(400).json({ success: false, message: error.message });
  }
};

exports.updateTask = async (req, res) => {
  try {
    const task = await Task.findByIdAndUpdate(
      req.params.id,
      req.body,
      { new: true, runValidators: true }
    );
    
    if (!task) {
      return res.status(404).json({ success: false, message: 'Task not found' });
    }
    
    res.status(200).json({ success: true, data: task });
  } catch (error) {
    res.status(400).json({ success: false, message: error.message });
  }
};
```

### MongoDB Models

```javascript
// models/User.js
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');

const UserSchema = new mongoose.Schema({
  username: { type: String, required: true, unique: true },
  email: { type: String, required: true, unique: true },
  password: { type: String, required: true, select: false },
  role: { type: String, enum: ['user', 'admin'], default: 'user' },
  department: String,
  status: { type: String, enum: ['active', 'inactive'], default: 'active' },
  lastLogin: Date,
  createdAt: { type: Date, default: Date.now }
});

UserSchema.pre('save', async function(next) {
  if (!this.isModified('password')) return next();
  this.password = await bcrypt.hash(this.password, 10);
});

UserSchema.methods.comparePassword = async function(candidatePassword) {
  return await bcrypt.compare(candidatePassword, this.password);
};

module.exports = mongoose.model('User', UserSchema);
```

```javascript
// models/Task.js
const mongoose = require('mongoose');

const TaskSchema = new mongoose.Schema({
  title: { type: String, required: true },
  description: String,
  status: { 
    type: String, 
    enum: ['todo', 'in_progress', 'done'], 
    default: 'todo' 
  },
  priority: { 
    type: String, 
    enum: ['low', 'medium', 'high', 'critical'], 
    default: 'medium' 
  },
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  createdBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  dueDate: Date,
  timeSpent: { type: Number, default: 0 }, // in minutes
  tags: [String],
  createdAt: { type: Date, default: Date.now },
  updatedAt: { type: Date, default: Date.now }
});

module.exports = mongoose.model('Task', TaskSchema);
```

## ML Service Code Examples (Python/FastAPI)

### Main FastAPI Application

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from typing import List, Dict, Optional
import joblib
import numpy as np
from datetime import datetime
import os

app = FastAPI(title="Enterprise User Management ML Service")

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Load or initialize models
MODEL_PATH = os.getenv('MODEL_PATH', './models')

class RiskPredictionRequest(BaseModel):
    userId: str
    features: Dict[str, float]

class AnomalyDetectionRequest(BaseModel):
    userId: str
    behavior: Dict[str, any]

class BurnoutPredictionRequest(BaseModel):
    userId: str
    workload: Dict[str, float]

class DelayPredictionRequest(BaseModel):
    projectId: str
    metrics: Dict[str, float]

@app.post("/predict-risk")
async def predict_risk(request: RiskPredictionRequest):
    """Predict user risk score based on behavior patterns"""
    try:
        # Extract features
        features = [
            request.features.get('loginAttempts', 0),
            request.features.get('taskCompletionRate', 1.0),
            request.features.get('avgResponseTime', 24),
            request.features.get('ticketCount', 0),
            request.features.get('failedLogins', 0)
        ]
        
        # Simple risk scoring logic (replace with trained model)
        risk_score = calculate_risk_score(features)
        
        return {
            "userId": request.userId,
            "riskScore": risk_score,
            "riskLevel": get_risk_level(risk_score),
            "recommendations": generate_risk_recommendations(risk_score)
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/detect-anomaly")
async def detect_anomaly(request: AnomalyDetectionRequest):
    """Detect anomalous user behavior"""
    try:
        behavior = request.behavior
        
        # Anomaly detection logic
        is_anomalous = False
        anomaly_score = 0.0
        reasons = []
        
        # Check login time
        login_hour = int(behavior.get('loginTime', '12:00').split(':')[0])
        if login_hour < 6 or login_hour > 22:
            is_anomalous = True
            anomaly_score += 0.3
            reasons.append("Unusual login time")
        
        # Check location
        if behavior.get('location') == 'unknown':
            is_anomalous = True
            anomaly_score += 0.4
            reasons.append("Unknown location")
        
        # Check activity pattern
        if behavior.get('activityPattern') == 'unusual':
            is_anomalous = True
            anomaly_score += 0.3
            reasons.append("Unusual activity pattern")
        
        return {
            "userId": request.userId,
            "isAnomalous": is_anomalous,
            "anomalyScore": min(anomaly_score, 1.0),
            "reasons": reasons,
            "timestamp": datetime.now().isoformat()
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/predict-burnout")
async def predict_burnout(request: BurnoutPredictionRequest):
    """Predict employee burnout risk"""
    try:
        workload = request.workload
        
        tasks_assigned = workload.get('tasksAssigned', 0)
        avg_work_hours = workload.get('avgWorkHours', 8)
        overtime_hours = workload.get('overtimeHours', 0)
        
        # Burnout score calculation
        burnout_score = 0.0
        
        if tasks_assigned > 10:
            burnout_score += 0.2 * (tasks_assigned - 10) / 10
        
        if avg_work_hours > 8:
            burnout_score += 0.3 * (avg_work_hours - 8) / 4
        
        if overtime_hours > 10:
            burnout_score += 0.5 * (overtime_hours / 40)
        
        burnout_score = min(burnout_score, 1.0)
        
        return {
            "userId": request.userId,
            "burnoutScore": burnout_score,
            "burnoutRisk": "high" if burnout_score > 0.7 else "medium" if burnout_score > 0.4 else "low",
            "recommendations": generate_burnout_recommendations(burnout_score),
            "suggestedActions": [
                "Redistribute tasks" if tasks_assigned > 12 else None,
                "Reduce overtime" if overtime_hours > 15 else None,
                "Mandatory break" if avg_work_hours > 10 else None
            ]
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/predict-delay")
async def predict_delay(request: DelayPredictionRequest):
    """Predict project delay probability"""
    try:
        metrics = request.metrics
        
        completion_rate = metrics.get('completionRate', 0.5)
        days_remaining = metrics.get('daysRemaining', 30)
        team_size = metrics.get('teamSize', 5)
        
        # Simple delay prediction
        expected_completion = completion_rate + (team_size * 0.05)
        delay_probability = max(0, 1 - expected_completion) * (30 / max(days_remaining, 1))
        delay_probability = min(delay_probability, 1.0)
        
        estimated_delay_days = 0
        if delay_probability > 0.5:
            estimated_delay_days = int(days_remaining * (1 - completion_rate) * 0.3)
        
        return {
            "projectId": request.projectId,
            "delayProbability": delay_probability,
            "estimatedDelayDays": estimated_delay_days,
            "confidenceLevel": 0.75,
            "recommendations": generate_delay_recommendations(delay_probability)
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

def calculate_risk_score(features: List[float]) -> float:
    """Calculate risk score from features"""
    login_attempts, completion_rate, response_time, ticket_count, failed_logins = features
    
    score = 0.0
    score += min(login_attempts / 10, 0.2)
    score += max(0, (1 - completion_rate) * 0.3)
    score += min(response_time / 72, 0.2)
    score += min(ticket_count / 20, 0.15)
    score += min(failed_logins / 5, 0.15)
    
    return min(score, 1.0)

def get_risk_level(score: float) -> str:
    if score > 0.7:
        return "high"
    elif score > 0.4:
        return "medium"
    return "low"

def generate_risk_recommendations(score: float) -> List[str]:
    recommendations = []
    if score > 0.7:
        recommendations.append("Immediate review required")
        recommendations.append("Restrict sensitive access")
    elif score > 0.4:
        recommendations.append("Monitor user activity closely")
    else:
        recommendations.append("Continue normal monitoring")
    return recommendations

def generate_burnout_recommendations(score: float) -> List[str]:
    if score > 0.7:
        return ["Immediate intervention needed", "Redistribute workload", "Consider time off"]
    elif score > 0.4:
        return ["Monitor workload", "Encourage breaks", "Review task assignments"]
    return ["Workload is manageable", "Maintain current pace"]

def generate_delay_recommendations(probability: float) -> List[str]:
    if probability > 0.7:
        return ["High delay risk - add resources", "Reassess timeline", "Prioritize critical tasks"]
    elif probability > 0.4:
        return ["Moderate risk - monitor closely", "Consider scope reduction"]
    return ["On track", "Continue current pace"]

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "ml-service"}
```

## Frontend Code Examples (React)

### User Dashboard Component

```javascript
// frontend/src/components/UserDashboard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const UserDashboard = () => {
  const [tasks, setTasks] = useState([]);
  const [stats, setStats] = useState({});
  const [burnoutRisk, setBurnoutRisk] = useState(null);
  const API_URL = process.env.REACT_APP_API_URL;
  const ML_API_URL = process.env.REACT_APP_ML_API_URL;

  useEffect(() => {
    fetchUserData();
  }, []);

  const fetchUserData = async () => {
    try {
      const token = localStorage.getItem('token');
      const config = {
        headers: { Authorization: `Bearer ${token}` }
      };

      // Fetch tasks
      const tasksRes = await axios.get(`${API_URL}/api/tasks`, config);
      setTasks(tasksRes.data.data);

      // Fetch user stats
      const statsRes = await axios.get(`${API_URL}/api/users/me/stats`, config);
      setStats(statsRes.data.data);

      // Check burnout risk
      const burnoutRes = await axios.post(
        `${ML_API_URL}/predict-burnout`,
        {
          userId: statsRes.data.data.userId,
          workload: {
            tasksAssigned: tasksRes.data.data.length,
            avgWorkHours: statsRes.data.data.avgWorkHours || 8,
            overtimeHours: statsRes.data.data.overtimeHours || 0
          }
        }
      );
      setBurnoutRisk(burnoutRes.data);
    } catch (error) {
      console.error('Error fetching user data:', error);
    }
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    try {
      const token = localStorage.getItem('token');
      await axios.put(
        `${API_URL}/api/tasks/${taskId}`,
        { status: newStatus },
        { headers: { Authorization: `Bearer ${token}` } }
      );
      fetchUserData();
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  return (
    <div className="user-dashboard">
      <h1>My Dashboard</h1>
      
      {burnoutRisk && burnoutRisk.burnoutRisk === 'high' && (
        <div className="alert alert-warning">
          <strong>Burnout Risk Alert:</strong> {burnoutRisk.recommendations[0]}
        </div>
      )}

      <div className="stats-grid">
        <div className="stat-card">
          <h3>Tasks Assigned</h3>
          <p>{tasks.length}</p>
        </div>
        <div className="stat-card">
          <h3>Completed</h3>
          <p>{tasks.filter(t => t.status === 'done').length}</p>
        </div>
        <div className="stat-card">
          <h3>In Progress</h3>
          <p>{tasks.filter(t => t.status === 'in_progress').length}</p>
        </div>
      </div>

      <div className="kanban-board">
        <div className="kanban-column">
          <h3>To Do</h3>
          {tasks.filter(t => t.status === 'todo').map(task => (
            <TaskCard 
              key={task._id} 
              task={task} 
              onStatusChange={updateTaskStatus}
            />
          ))}
        </div>
        
        <div className="kanban-column">
          <h3>In Progress</h3>
          {tasks.filter(t => t.status === 'in_progress').map(task => (
            <TaskCard 
              key={task._id} 
              task={task} 
              onStatusChange={updateTaskStatus}
            />
          ))}
        </div>
        
        <div className="kanban-column">
          <h3>Done</h3>
          {tasks.filter(t => t.status === 'done').map(task => (
            <TaskCard 
              key={task._id} 
              task={task} 
              onStatusChange={updateTaskStatus}
            />
          ))}
        </div>
      </div>
    </div>
  );
};

const TaskCard = ({ task, onStatusChange }) => {
  return (
    <div className={`task-card priority-${task.priority}`}>
      <h4>{task.title}</h4>
      <p>{task.description}</p>
      <div className="task-actions">
        {task.status === 'todo' && (
          <button onClick={() => onStatusChange(task._id, 'in_progress')}>
            Start
          </button>
        )}
        {task.status === 'in_progress' && (
          <button onClick={() => onStatusChange(task._id, 'done')}>
            Complete
          </button>
        )}
      </div>
    </div>
  );
};

export default UserDashboard;
```

### Admin Analytics Dashboard

```javascript
// frontend/src/components/AdminDashboard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const AdminDashboard = () => {
  const [users, setUsers] = useState([]);
  const [riskUsers, setRiskUsers] = useState([]);
  const [anomalies, setAnomalies] = useState([]);
  const API_URL = process.env.REACT_APP_API_URL;
  const ML_API_URL = process.env.REACT_APP_ML_API_URL;

  useEffect(() => {
    fetchAdminData();
  }, []);

  const fetchAdminData = async () => {
    try {
      const token = localStorage.getItem('token');
      const config = {
        headers: { Authorization: `Bearer ${token}` }
      };

      // Fetch all users
      const usersRes = await axios.get(`${API_URL}/api/users`, config);
      setUsers(usersRes.data.data);

      // Analyze each user for risk
      const riskAnalysis = await Promise.all(
        usersRes.data.data.map(async (user) => {
          try {
            const riskRes = await axios.post(
              `${ML_API_URL}/predict-risk`,
              {
                userId: user._id,
                features: {
                  loginAttempts: user.loginAttempts || 0,
                  taskCompletionRate: user.taskCompletionRate || 1.0,
                  avgResponseTime: user.avgResponseTime || 24,
                  ticketCount: user.ticketCount || 0,
                  failedLogins: user.failedLogins || 0
                }
              }
            );
            return { ...user, risk: riskRes.data };
          } catch (error) {
            return { ...user, risk: null };
          }
        })
      );

      const highRiskUsers = riskAnalysis.filter(
        u => u.risk && u.risk.riskLevel === 'high'
      );
      setRiskUsers(highRiskUsers);
    } catch (error) {
      console.error('Error fetching admin data:', error);
    }
  };

  return (
    <div className="admin-dashboard">
      <h1>Admin Dashboard</h1>

      <div className="stats-overview">
        <div className="stat-card">
          <h3>Total Users</h3>
          <p>{users.length}</p>
        </div>
        <div className="stat-card alert">
          <h3>High Risk Users</h3>
          <p>{riskUsers.length}</p>
        </div>
      </div>

      {riskUsers.length > 0 && (
        <div className="risk-alerts">
          <h2>Risk Alerts</h2>
          {riskUsers.map(user => (
            <div key={user._id} className="risk-alert-card">
              <h4>{user.username}</h4>
              <p>Risk Score: {user.risk.riskScore.toFixed(2)}</p>
              <p>Level: {user.risk.riskLevel}</p>
              <ul>
                {user.risk.recommendations.map((rec, idx) => (
                  <li key={idx}>{rec}</li>
                ))}
              </ul>
            </div>
          ))}
        </div>
      )}

      <div className="user-management">
        <h2>User Management</h2>
        <table>
          <thead>
            <tr>
              <th>Username</th>
              <th>Email</th>
              <th>Role</th>
              <th>Status</th>
              <th>Actions</th>
            </tr>
          </thead>
          <tbody>
            {users.map(user => (
              <tr key={user._id}>
                <td>{user.username}</td>
                <td>{user.email}</td>
                <td>{user.role}</td>
                <td>{user.status}</td>
                <td>
                  <button>Edit</button>
                  <button>Delete</button>
                </td>
              </tr>
            ))}
          </tbody>
        </table>
      </div>
    </div>
  );
};

export default AdminDashboard;
```

## Configuration

### Environment Variables

**Backend (.env)**
```bash
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
JWT_SECRET=your_jwt_secret_key_min_32_chars
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
```

**ML Service (.env)**
```bash
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
MODEL_PATH=./models
LOG_LEVEL=info
PREDICTION_THRESHOLD=0.7
```

**Frontend (.env)**
```bash
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
```

## Common Patterns

### Protecting Routes with JWT

```javascript
// Example protected route
const express = require('express');
const router = express.Router();
const { protect, authorize } = require('../middleware/auth');

// User-only route
router.get('/api/tasks/my-tasks', protect, async (req, res) => {
  // req.user is populated by protect middleware
  const tasks = await Task.find({ assignedTo: req.user.id });
  res.json({ success: true, data: tasks });
});

// Admin-only route
router.post('/api/users', protect, authorize('admin'), async (req, res) => {
  const user = await User.create(req.body);
  res.json({ success: true, data: user });
});
```

### Integrating ML Predictions in Workflows

```javascript
// Automatically check user risk on login
exports.login = async (req, res)
