---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics, risk detection, and task management
triggers:
  - "set up enterprise user management system"
  - "implement AI-powered user analytics"
  - "create user management dashboard with task tracking"
  - "add risk detection and anomaly detection"
  - "build admin panel with user role management"
  - "integrate AI ticket classification system"
  - "develop kanban board with time tracking"
  - "implement burnout detection for employees"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System with AI Analytics is a full-stack application that combines user/task management with intelligent insights. It provides:

- **User Management**: Secure authentication, role-based access control, user CRUD operations
- **Task Management**: Kanban boards, time tracking, task assignment
- **Support System**: Ticket management with AI-powered classification and routing
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, project delay prediction
- **Admin Dashboard**: Organization analytics, audit logs, performance metrics

**Stack**: React.js frontend, Node.js/Express backend, FastAPI ML service, MongoDB database, JWT authentication

## Installation

### Prerequisites

- Node.js 14+ and npm
- Python 3.8+
- MongoDB (local or Atlas)

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

Create `.env` file in `backend/`:

```env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
JWT_SECRET=your_jwt_secret_key
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
```

Start backend:

```bash
npm start
# or for development
npm run dev
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt
```

Create `.env` file in `ml-service/`:

```env
MODEL_PATH=./models
LOG_LEVEL=INFO
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
```

Start ML service:

```bash
uvicorn main:app --reload --port 8000
```

### Frontend Setup

```bash
cd frontend
npm install
```

Create `.env` file in `frontend/`:

```env
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_URL=http://localhost:8000
```

Start frontend:

```bash
npm start
```

Access application at `http://localhost:3000`

## Key API Endpoints

### Authentication (Backend)

```javascript
// POST /api/auth/register
{
  "username": "john.doe",
  "email": "john@company.com",
  "password": "securePassword123",
  "role": "user" // or "admin"
}

// POST /api/auth/login
{
  "email": "john@company.com",
  "password": "securePassword123"
}
// Response: { "token": "jwt_token", "user": {...} }
```

### User Management (Backend)

```javascript
// GET /api/users - List all users (admin only)
// GET /api/users/:id - Get user details
// PUT /api/users/:id - Update user
// DELETE /api/users/:id - Delete user (admin only)

// Headers for authenticated requests
{
  "Authorization": "Bearer <JWT_TOKEN>"
}
```

### Task Management (Backend)

```javascript
// POST /api/tasks - Create task
{
  "title": "Implement feature X",
  "description": "Add new authentication flow",
  "assignedTo": "user_id",
  "status": "todo", // todo, in_progress, done
  "priority": "high",
  "dueDate": "2026-05-01"
}

// GET /api/tasks - Get all tasks
// GET /api/tasks/user/:userId - Get user tasks
// PUT /api/tasks/:id - Update task status
// DELETE /api/tasks/:id - Delete task
```

### Support Tickets (Backend)

```javascript
// POST /api/tickets - Create ticket
{
  "title": "Cannot access dashboard",
  "description": "Getting 403 error when trying to view analytics",
  "category": "technical", // technical, billing, general
  "priority": "medium"
}

// GET /api/tickets - Get all tickets
// PUT /api/tickets/:id - Update ticket
```

### AI Analytics Endpoints (ML Service)

```python
# POST /predict/risk - Predict user risk score
{
  "user_id": "user123",
  "login_frequency": 45,
  "failed_login_attempts": 2,
  "task_completion_rate": 0.85,
  "avg_task_duration": 3.5
}
# Response: { "risk_score": 0.23, "risk_level": "low" }

# POST /predict/burnout - Detect employee burnout
{
  "user_id": "user123",
  "tasks_assigned": 15,
  "tasks_completed": 10,
  "avg_working_hours": 9.5,
  "overtime_hours": 12,
  "leave_days": 2
}
# Response: { "burnout_probability": 0.67, "burnout_risk": "high" }

# POST /classify/ticket - Classify support ticket
{
  "title": "Password reset not working",
  "description": "I clicked reset password but no email received"
}
# Response: { "category": "technical", "priority": "high", "suggested_department": "IT Support" }

# POST /predict/project-delay - Predict project delays
{
  "project_id": "proj123",
  "total_tasks": 50,
  "completed_tasks": 20,
  "days_elapsed": 30,
  "deadline_days": 60,
  "team_size": 5
}
# Response: { "delay_probability": 0.45, "estimated_completion_days": 65 }

# POST /detect/anomaly - Detect anomalous behavior
{
  "user_id": "user123",
  "login_time": "03:45:00",
  "ip_address": "192.168.1.100",
  "actions_per_session": 150,
  "data_accessed_mb": 500
}
# Response: { "is_anomaly": true, "anomaly_score": 0.89, "reasons": ["unusual_login_time", "high_data_access"] }
```

## Code Examples

### Backend - User Authentication (Node.js/Express)

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');

const authMiddleware = (req, res, next) => {
  try {
    const token = req.header('Authorization')?.replace('Bearer ', '');
    
    if (!token) {
      return res.status(401).json({ error: 'Authentication required' });
    }
    
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

```javascript
// backend/routes/users.js
const express = require('express');
const router = express.Router();
const { authMiddleware, adminOnly } = require('../middleware/auth');
const User = require('../models/User');

// Get all users (admin only)
router.get('/', authMiddleware, adminOnly, async (req, res) => {
  try {
    const users = await User.find().select('-password');
    res.json(users);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Get user profile
router.get('/:id', authMiddleware, async (req, res) => {
  try {
    const user = await User.findById(req.params.id).select('-password');
    if (!user) {
      return res.status(404).json({ error: 'User not found' });
    }
    res.json(user);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Update user
router.put('/:id', authMiddleware, async (req, res) => {
  try {
    const { username, email, role } = req.body;
    const user = await User.findByIdAndUpdate(
      req.params.id,
      { username, email, role },
      { new: true }
    ).select('-password');
    res.json(user);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

module.exports = router;
```

### Backend - Task Management

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
    enum: ['todo', 'in_progress', 'done'],
    default: 'todo'
  },
  priority: {
    type: String,
    enum: ['low', 'medium', 'high'],
    default: 'medium'
  },
  dueDate: { type: Date },
  timeTracked: { type: Number, default: 0 }, // in minutes
  createdAt: { type: Date, default: Date.now },
  updatedAt: { type: Date, default: Date.now }
});

module.exports = mongoose.model('Task', taskSchema);
```

```javascript
// backend/routes/tasks.js
const express = require('express');
const router = express.Router();
const { authMiddleware } = require('../middleware/auth');
const Task = require('../models/Task');

// Create task
router.post('/', authMiddleware, async (req, res) => {
  try {
    const task = new Task({
      ...req.body,
      createdBy: req.user.id
    });
    await task.save();
    res.status(201).json(task);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Get user tasks
router.get('/user/:userId', authMiddleware, async (req, res) => {
  try {
    const tasks = await Task.find({ assignedTo: req.params.userId })
      .populate('assignedTo', 'username email')
      .populate('createdBy', 'username');
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Update task status
router.put('/:id', authMiddleware, async (req, res) => {
  try {
    const task = await Task.findByIdAndUpdate(
      req.params.id,
      { ...req.body, updatedAt: Date.now() },
      { new: true }
    );
    res.json(task);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

module.exports = router;
```

### ML Service - AI Analytics (FastAPI)

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
import numpy as np
from sklearn.ensemble import RandomForestClassifier, IsolationForest
import pickle
import os

app = FastAPI(title="Enterprise AI Analytics Service")

# Models (load or initialize)
MODEL_PATH = os.getenv('MODEL_PATH', './models')

class RiskPredictionInput(BaseModel):
    user_id: str
    login_frequency: int
    failed_login_attempts: int
    task_completion_rate: float
    avg_task_duration: float

class BurnoutInput(BaseModel):
    user_id: str
    tasks_assigned: int
    tasks_completed: int
    avg_working_hours: float
    overtime_hours: float
    leave_days: int

class TicketInput(BaseModel):
    title: str
    description: str

class ProjectDelayInput(BaseModel):
    project_id: str
    total_tasks: int
    completed_tasks: int
    days_elapsed: int
    deadline_days: int
    team_size: int

class AnomalyInput(BaseModel):
    user_id: str
    login_time: str
    ip_address: str
    actions_per_session: int
    data_accessed_mb: float

@app.post("/predict/risk")
async def predict_risk(data: RiskPredictionInput):
    """Predict user risk score based on behavioral patterns"""
    try:
        # Feature engineering
        features = np.array([[
            data.login_frequency,
            data.failed_login_attempts,
            data.task_completion_rate,
            data.avg_task_duration
        ]])
        
        # Simple risk calculation (in production, use trained model)
        risk_score = (
            (data.failed_login_attempts * 0.3) +
            ((1 - data.task_completion_rate) * 0.4) +
            (min(data.avg_task_duration / 10, 1) * 0.3)
        )
        
        risk_level = "low" if risk_score < 0.3 else "medium" if risk_score < 0.7 else "high"
        
        return {
            "user_id": data.user_id,
            "risk_score": round(risk_score, 2),
            "risk_level": risk_level,
            "factors": {
                "failed_logins": data.failed_login_attempts > 3,
                "low_completion": data.task_completion_rate < 0.7,
                "slow_performance": data.avg_task_duration > 5
            }
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/predict/burnout")
async def predict_burnout(data: BurnoutInput):
    """Detect employee burnout risk"""
    try:
        # Calculate burnout indicators
        task_completion_rate = data.tasks_completed / data.tasks_assigned if data.tasks_assigned > 0 else 0
        overtime_ratio = data.overtime_hours / (data.avg_working_hours * 30) if data.avg_working_hours > 0 else 0
        
        burnout_score = (
            (1 - task_completion_rate) * 0.3 +
            (overtime_ratio * 0.4) +
            ((10 - data.leave_days) / 10 * 0.3)
        )
        
        risk = "low" if burnout_score < 0.4 else "medium" if burnout_score < 0.7 else "high"
        
        return {
            "user_id": data.user_id,
            "burnout_probability": round(burnout_score, 2),
            "burnout_risk": risk,
            "recommendations": [
                "Reduce overtime hours" if overtime_ratio > 0.2 else None,
                "Encourage taking leave" if data.leave_days < 5 else None,
                "Redistribute workload" if task_completion_rate < 0.6 else None
            ]
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/classify/ticket")
async def classify_ticket(data: TicketInput):
    """Classify support ticket using NLP"""
    try:
        text = f"{data.title} {data.description}".lower()
        
        # Simple keyword-based classification (use NLP model in production)
        if any(word in text for word in ['password', 'login', 'access', 'error', 'bug']):
            category = "technical"
            priority = "high"
            department = "IT Support"
        elif any(word in text for word in ['payment', 'invoice', 'billing', 'charge']):
            category = "billing"
            priority = "medium"
            department = "Finance"
        else:
            category = "general"
            priority = "low"
            department = "General Support"
        
        return {
            "category": category,
            "priority": priority,
            "suggested_department": department,
            "confidence": 0.85
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/predict/project-delay")
async def predict_project_delay(data: ProjectDelayInput):
    """Predict if project will be delayed"""
    try:
        completion_rate = data.completed_tasks / data.total_tasks if data.total_tasks > 0 else 0
        expected_completion_rate = data.days_elapsed / data.deadline_days
        
        delay_indicator = expected_completion_rate - completion_rate
        delay_probability = min(max(delay_indicator, 0), 1)
        
        estimated_days = data.deadline_days * (1 + delay_indicator)
        
        return {
            "project_id": data.project_id,
            "delay_probability": round(delay_probability, 2),
            "estimated_completion_days": int(estimated_days),
            "on_track": delay_probability < 0.3,
            "suggested_actions": [
                "Increase team size" if delay_probability > 0.5 else None,
                "Reprioritize tasks" if completion_rate < 0.4 else None
            ]
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/detect/anomaly")
async def detect_anomaly(data: AnomalyInput):
    """Detect anomalous user behavior"""
    try:
        # Parse login time
        hour = int(data.login_time.split(':')[0])
        
        anomalies = []
        score = 0
        
        # Check for unusual login time (outside 6 AM - 10 PM)
        if hour < 6 or hour > 22:
            anomalies.append("unusual_login_time")
            score += 0.3
        
        # Check for excessive actions
        if data.actions_per_session > 100:
            anomalies.append("high_action_count")
            score += 0.2
        
        # Check for high data access
        if data.data_accessed_mb > 200:
            anomalies.append("high_data_access")
            score += 0.4
        
        is_anomaly = score > 0.5
        
        return {
            "user_id": data.user_id,
            "is_anomaly": is_anomaly,
            "anomaly_score": round(score, 2),
            "reasons": anomalies,
            "action_required": "investigate" if is_anomaly else "monitor"
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "AI Analytics"}
```

### Frontend - User Dashboard (React)

```javascript
// frontend/src/components/UserDashboard.js
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const UserDashboard = () => {
  const [tasks, setTasks] = useState([]);
  const [loading, setLoading] = useState(true);
  const [burnoutData, setBurnoutData] = useState(null);

  const API_URL = process.env.REACT_APP_API_URL;
  const ML_URL = process.env.REACT_APP_ML_URL;

  useEffect(() => {
    fetchTasks();
    fetchBurnoutAnalysis();
  }, []);

  const fetchTasks = async () => {
    try {
      const token = localStorage.getItem('token');
      const userId = localStorage.getItem('userId');
      
      const response = await axios.get(`${API_URL}/tasks/user/${userId}`, {
        headers: { Authorization: `Bearer ${token}` }
      });
      
      setTasks(response.data);
      setLoading(false);
    } catch (error) {
      console.error('Error fetching tasks:', error);
      setLoading(false);
    }
  };

  const fetchBurnoutAnalysis = async () => {
    try {
      const userId = localStorage.getItem('userId');
      const tasksAssigned = tasks.filter(t => t.status !== 'done').length;
      const tasksCompleted = tasks.filter(t => t.status === 'done').length;
      
      const response = await axios.post(`${ML_URL}/predict/burnout`, {
        user_id: userId,
        tasks_assigned: tasksAssigned,
        tasks_completed: tasksCompleted,
        avg_working_hours: 8.5,
        overtime_hours: 10,
        leave_days: 3
      });
      
      setBurnoutData(response.data);
    } catch (error) {
      console.error('Error fetching burnout analysis:', error);
    }
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    try {
      const token = localStorage.getItem('token');
      
      await axios.put(`${API_URL}/tasks/${taskId}`, 
        { status: newStatus },
        { headers: { Authorization: `Bearer ${token}` }}
      );
      
      fetchTasks();
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  const getTasksByStatus = (status) => {
    return tasks.filter(task => task.status === status);
  };

  return (
    <div className="dashboard">
      <h1>My Dashboard</h1>
      
      {burnoutData && burnoutData.burnout_risk === 'high' && (
        <div className="alert alert-warning">
          ⚠️ High burnout risk detected. Consider taking a break or discussing workload with your manager.
        </div>
      )}
      
      <div className="kanban-board">
        <div className="kanban-column">
          <h3>To Do ({getTasksByStatus('todo').length})</h3>
          {getTasksByStatus('todo').map(task => (
            <div key={task._id} className="task-card">
              <h4>{task.title}</h4>
              <p>{task.description}</p>
              <span className={`priority-badge ${task.priority}`}>
                {task.priority}
              </span>
              <button onClick={() => updateTaskStatus(task._id, 'in_progress')}>
                Start →
              </button>
            </div>
          ))}
        </div>
        
        <div className="kanban-column">
          <h3>In Progress ({getTasksByStatus('in_progress').length})</h3>
          {getTasksByStatus('in_progress').map(task => (
            <div key={task._id} className="task-card active">
              <h4>{task.title}</h4>
              <p>{task.description}</p>
              <button onClick={() => updateTaskStatus(task._id, 'done')}>
                Complete ✓
              </button>
            </div>
          ))}
        </div>
        
        <div className="kanban-column">
          <h3>Done ({getTasksByStatus('done').length})</h3>
          {getTasksByStatus('done').map(task => (
            <div key={task._id} className="task-card completed">
              <h4>{task.title}</h4>
              <p>{task.description}</p>
              <span className="completed-badge">✓ Completed</span>
            </div>
          ))}
        </div>
      </div>
    </div>
  );
};

export default UserDashboard;
```

### Frontend - Admin Analytics

```javascript
// frontend/src/components/AdminAnalytics.js
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const AdminAnalytics = () => {
  const [users, setUsers] = useState([]);
  const [riskAnalysis, setRiskAnalysis] = useState({});
  
  const API_URL = process.env.REACT_APP_API_URL;
  const ML_URL = process.env.REACT_APP_ML_URL;

  useEffect(() => {
    fetchUsers();
  }, []);

  const fetchUsers = async () => {
    try {
      const token = localStorage.getItem('token');
      const response = await axios.get(`${API_URL}/users`, {
        headers: { Authorization: `Bearer ${token}` }
      });
      
      setUsers(response.data);
      
      // Fetch risk analysis for each user
      response.data.forEach(user => {
        analyzeUserRisk(user._id);
      });
    } catch (error) {
      console.error('Error fetching users:', error);
    }
  };

  const analyzeUserRisk = async (userId) => {
    try {
      const response = await axios.post(`${ML_URL}/predict/risk`, {
        user_id: userId,
        login_frequency: 30,
        failed_login_attempts: Math.floor(Math.random() * 5),
        task_completion_rate: 0.7 + Math.random() * 0.3,
        avg_task_duration: 2 + Math.random() * 4
      });
      
      setRiskAnalysis(prev => ({
        ...prev,
        [userId]: response.data
      }));
    } catch (error) {
      console.error('Error analyzing user risk:', error);
    }
  };

  return (
    <div className="admin-analytics">
      <h1>User Analytics Dashboard</h1>
      
      <div className="user-table">
        <table>
          <thead>
            <tr>
              <th>User</th>
              <th>Email</th>
              <th>Role</th>
              <th>Risk Level</th>
              <th>Risk Score</th>
              <th>Actions</th>
            </tr>
          </thead>
          <tbody>
            {users.map(user => (
              <tr key={user._id}>
                <td>{user.username}</td>
                <td>{user.email}</td>
                <td>{user.role}</td>
                <td>
                  {riskAnalysis[user._id] && (
                    <span className={`risk-badge ${riskAnalysis[user._id].risk_level}`}>
                      {riskAnalysis[user._id].risk_level}
                    </span>
                  )}
                </td>
                <td>
                  {riskAnalysis[user._id]?.risk_score || 'Loading...'}
                </td>
                <td>
                  <button onClick={() => analyzeUserRisk(user._id)}>
                    Refresh Analysis
                  </button>
                </td>
              </tr>
            ))}
          </tbody>
        </table>
      </div>
    </div>
  );
};

export default AdminAnalytics;
```

## Common Patterns

### Integrating AI Predictions in Workflows

```javascript
// backend/routes/tickets.js - Auto-classify tickets on creation
router.post('/', authMiddleware, async (req, res) => {
  try {
    const { title, description } = req.body;
    
    // Call ML service for classification
    const mlResponse = await axios.post(`${process.env.ML_SERVICE_URL}/classify/ticket`, {
      title,
      description
    });
    
    const ticket = new Ticket({
      title,
      description,
      category: mlResponse.data.category,
      priority: mlResponse.data.priority,
      assignedDepartment: mlResponse.data.suggested_department,
      createdBy: req.user.id
    });
    
    await ticket.save();
    res.status(201).json(ticket);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});
```

### Scheduled Risk Analysis

```javascript
// backend/jobs/riskAnalysis.js
const cron = require('node-cron');
const axios = require('axios');
const User = require('../models/User');
const Task = require('../models/Task');

// Run daily at 2 AM
cron.schedule('0 2 * * *', async () => {
  console.log('Running daily risk analysis...');
  
  const users = await User.find();
  
  for (const user of users) {
    const tasks = await Task.find({ assignedTo: user._id });
    const completedTasks = tasks.filter(t => t.status === 'done');
    
    const riskData = {
      user_id: user._id,
      login_frequency: user.loginCount || 0,
      failed_login_attempts: user.failedLogins || 0,
      task_completion_rate: completedTasks.length / tasks.length || 0,
      avg_task_duration: calculateAvgDuration(tasks)
    };
    
    try {
      const response = await axios.post(
        `${process.env.ML_SERVICE_URL}/predict/risk`,
        riskData
      );
      
      // Store or alert based on risk level
      if (response.data.risk_level === 'high') {
        // Send alert to admin
        console.log(`High risk detected for user ${user.username}`);
      }
    } catch (error) {
      console.error(`Risk analysis failed for user ${user._id}:`, error);
    }
  }
});

function calculateAvgDuration(tasks) {
  const durations = tasks.map(t => t.timeTracked || 0);
  return durations.reduce((a, b) => a + b, 0) / durations.length || 0;
}
```

## Configuration

### Environment Variables

**Backend (.env)**:
```env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
JWT_SECRET=your_secure_random_secret
JWT_EXPIRES_IN=7d
ML_SERVICE_URL=http://localhost:8000
FRONTEND_URL=http://localhost:3000
NODE_ENV=development
```

**ML Service (.env)**:
```env
MODEL_PATH=./models
LOG_LEVEL=INFO
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
RISK_THRESHOLD_HIGH=0.7
RISK_THRESHOLD_MEDIUM=0.4
BURNOUT_THRESHOLD_HIGH=0.7
```

**Frontend (.env)**:
```env
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_URL=http://localhost:8000
REACT_APP_SOCKET_URL=http://localhost:5000
```

### MongoDB Indexes

```javascript
