---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics, task tracking, and automated ticket classification
triggers:
  - "set up enterprise user management system"
  - "implement AI analytics for user management"
  - "create task tracking with kanban board"
  - "build admin dashboard with user analytics"
  - "add AI ticket classification system"
  - "integrate burnout detection and risk prediction"
  - "develop user management with role-based access"
  - "setup ML service for anomaly detection"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI coding agents to work with an enterprise-grade user management system that combines traditional CRUD operations with AI-powered analytics. The system includes user authentication, task management with Kanban boards, support ticket handling, and ML-based insights for risk detection, anomaly detection, burnout analysis, and predictive analytics.

## Project Overview

The Enterprise User Management System is a three-tier application:
- **Frontend**: React.js dashboard for users and admins
- **Backend**: Node.js REST API with JWT authentication
- **ML Service**: FastAPI-based machine learning service with scikit-learn and River

Key capabilities:
- User and role management with JWT authentication
- Task tracking with Kanban workflow (To Do → In Progress → Done)
- Support ticket system with AI-powered classification
- AI analytics: risk prediction, anomaly detection, burnout analysis
- Time tracking and performance insights
- Audit logging and security monitoring

## Installation & Setup

### Prerequisites
- Node.js 14+ and npm
- Python 3.8+
- MongoDB running locally or remote connection string

### Clone and Install

```bash
# Clone repository
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics

# Install backend dependencies
cd backend
npm install

# Install ML service dependencies
cd ../ml-service
pip install -r requirements.txt

# Install frontend dependencies
cd ../frontend
npm install
```

### Environment Configuration

**Backend (.env)**:
```bash
# backend/.env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
JWT_SECRET=your_jwt_secret_key_here
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
```

**ML Service (.env)**:
```bash
# ml-service/.env
MODEL_PATH=./models
LOG_LEVEL=INFO
BACKEND_URL=http://localhost:5000
```

**Frontend (.env)**:
```bash
# frontend/.env
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_URL=http://localhost:8000
```

### Running the Application

```bash
# Terminal 1: Start Backend
cd backend
npm start

# Terminal 2: Start ML Service
cd ml-service
uvicorn main:app --reload --port 8000

# Terminal 3: Start Frontend
cd frontend
npm start
```

Access points:
- Frontend: http://localhost:3000
- Backend API: http://localhost:5000
- ML Service: http://localhost:8000/docs (FastAPI Swagger UI)

## Backend API Reference

### Authentication

```javascript
// POST /api/auth/register
const registerUser = async (userData) => {
  const response = await fetch('http://localhost:5000/api/auth/register', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      username: userData.username,
      email: userData.email,
      password: userData.password,
      role: userData.role || 'user'
    })
  });
  return response.json();
};

// POST /api/auth/login
const loginUser = async (credentials) => {
  const response = await fetch('http://localhost:5000/api/auth/login', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      email: credentials.email,
      password: credentials.password
    })
  });
  const data = await response.json();
  // Store JWT token
  localStorage.setItem('token', data.token);
  return data;
};
```

### User Management (Admin)

```javascript
// GET /api/users - Get all users
const getAllUsers = async (token) => {
  const response = await fetch('http://localhost:5000/api/users', {
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return response.json();
};

// PUT /api/users/:id - Update user
const updateUser = async (userId, updates, token) => {
  const response = await fetch(`http://localhost:5000/api/users/${userId}`, {
    method: 'PUT',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify(updates)
  });
  return response.json();
};

// DELETE /api/users/:id - Delete user
const deleteUser = async (userId, token) => {
  const response = await fetch(`http://localhost:5000/api/users/${userId}`, {
    method: 'DELETE',
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return response.json();
};
```

### Task Management

```javascript
// POST /api/tasks - Create task
const createTask = async (taskData, token) => {
  const response = await fetch('http://localhost:5000/api/tasks', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify({
      title: taskData.title,
      description: taskData.description,
      assignedTo: taskData.userId,
      priority: taskData.priority,
      dueDate: taskData.dueDate,
      status: 'todo' // todo, in_progress, done
    })
  });
  return response.json();
};

// GET /api/tasks/user/:userId - Get user tasks
const getUserTasks = async (userId, token) => {
  const response = await fetch(`http://localhost:5000/api/tasks/user/${userId}`, {
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return response.json();
};

// PATCH /api/tasks/:id/status - Update task status
const updateTaskStatus = async (taskId, newStatus, token) => {
  const response = await fetch(`http://localhost:5000/api/tasks/${taskId}/status`, {
    method: 'PATCH',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify({ status: newStatus })
  });
  return response.json();
};
```

### Support Tickets

```javascript
// POST /api/tickets - Create ticket
const createTicket = async (ticketData, token) => {
  const response = await fetch('http://localhost:5000/api/tickets', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify({
      title: ticketData.title,
      description: ticketData.description,
      category: ticketData.category,
      priority: ticketData.priority
    })
  });
  return response.json();
};

// GET /api/tickets - Get all tickets (admin)
const getAllTickets = async (token) => {
  const response = await fetch('http://localhost:5000/api/tickets', {
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return response.json();
};
```

## ML Service API Reference

### AI Ticket Classification

```javascript
// POST /classify-ticket
const classifyTicket = async (ticketText) => {
  const response = await fetch('http://localhost:8000/classify-ticket', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      text: ticketText,
      title: "Support Request"
    })
  });
  const data = await response.json();
  // Returns: { category: 'technical', priority: 'high', suggested_assignee: 'dept_id' }
  return data;
};
```

### Risk Prediction

```javascript
// POST /predict-risk
const predictUserRisk = async (userBehavior) => {
  const response = await fetch('http://localhost:8000/predict-risk', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      user_id: userBehavior.userId,
      login_frequency: userBehavior.loginFrequency,
      failed_logins: userBehavior.failedLogins,
      task_completion_rate: userBehavior.taskCompletionRate,
      avg_response_time: userBehavior.avgResponseTime
    })
  });
  const data = await response.json();
  // Returns: { risk_score: 0.75, risk_level: 'high', factors: [...] }
  return data;
};
```

### Anomaly Detection

```javascript
// POST /detect-anomaly
const detectAnomaly = async (activityLog) => {
  const response = await fetch('http://localhost:8000/detect-anomaly', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      user_id: activityLog.userId,
      timestamp: activityLog.timestamp,
      action_type: activityLog.actionType,
      resource_accessed: activityLog.resource,
      ip_address: activityLog.ipAddress
    })
  });
  const data = await response.json();
  // Returns: { is_anomaly: true, confidence: 0.89, anomaly_type: 'unusual_access' }
  return data;
};
```

### Burnout Detection

```javascript
// POST /detect-burnout
const detectBurnout = async (userWorkload) => {
  const response = await fetch('http://localhost:8000/detect-burnout', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      user_id: userWorkload.userId,
      tasks_assigned: userWorkload.tasksAssigned,
      tasks_completed: userWorkload.tasksCompleted,
      avg_hours_worked: userWorkload.avgHoursWorked,
      overtime_hours: userWorkload.overtimeHours,
      missed_deadlines: userWorkload.missedDeadlines
    })
  });
  const data = await response.json();
  // Returns: { burnout_risk: 'high', score: 0.82, recommendations: [...] }
  return data;
};
```

### Predictive Project Insights

```javascript
// POST /predict-project-delay
const predictProjectDelay = async (projectData) => {
  const response = await fetch('http://localhost:8000/predict-project-delay', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      project_id: projectData.projectId,
      total_tasks: projectData.totalTasks,
      completed_tasks: projectData.completedTasks,
      team_size: projectData.teamSize,
      avg_task_completion_time: projectData.avgCompletionTime,
      days_remaining: projectData.daysRemaining
    })
  });
  const data = await response.json();
  // Returns: { delay_probability: 0.65, estimated_delay_days: 5, risk_factors: [...] }
  return data;
};
```

## React Frontend Patterns

### Protected Routes with JWT

```javascript
// src/components/ProtectedRoute.js
import React from 'react';
import { Navigate } from 'react-router-dom';

const ProtectedRoute = ({ children, requireAdmin = false }) => {
  const token = localStorage.getItem('token');
  const userRole = localStorage.getItem('role');
  
  if (!token) {
    return <Navigate to="/login" />;
  }
  
  if (requireAdmin && userRole !== 'admin') {
    return <Navigate to="/dashboard" />;
  }
  
  return children;
};

export default ProtectedRoute;
```

### Kanban Board Component

```javascript
// src/components/KanbanBoard.js
import React, { useState, useEffect } from 'react';

const KanbanBoard = ({ userId }) => {
  const [tasks, setTasks] = useState({ todo: [], in_progress: [], done: [] });
  const token = localStorage.getItem('token');

  useEffect(() => {
    fetchTasks();
  }, [userId]);

  const fetchTasks = async () => {
    const response = await fetch(`${process.env.REACT_APP_API_URL}/tasks/user/${userId}`, {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    const data = await response.json();
    
    const grouped = {
      todo: data.filter(t => t.status === 'todo'),
      in_progress: data.filter(t => t.status === 'in_progress'),
      done: data.filter(t => t.status === 'done')
    };
    setTasks(grouped);
  };

  const moveTask = async (taskId, newStatus) => {
    await fetch(`${process.env.REACT_APP_API_URL}/tasks/${taskId}/status`, {
      method: 'PATCH',
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
      {['todo', 'in_progress', 'done'].map(status => (
        <div key={status} className="kanban-column">
          <h3>{status.replace('_', ' ').toUpperCase()}</h3>
          {tasks[status].map(task => (
            <div key={task._id} className="task-card">
              <h4>{task.title}</h4>
              <p>{task.description}</p>
              <select 
                value={task.status} 
                onChange={(e) => moveTask(task._id, e.target.value)}
              >
                <option value="todo">To Do</option>
                <option value="in_progress">In Progress</option>
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

### Admin Dashboard with AI Insights

```javascript
// src/pages/AdminDashboard.js
import React, { useState, useEffect } from 'react';

const AdminDashboard = () => {
  const [users, setUsers] = useState([]);
  const [analytics, setAnalytics] = useState(null);
  const token = localStorage.getItem('token');

  useEffect(() => {
    fetchUsers();
    fetchAnalytics();
  }, []);

  const fetchUsers = async () => {
    const response = await fetch(`${process.env.REACT_APP_API_URL}/users`, {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    const data = await response.json();
    setUsers(data);
  };

  const fetchAnalytics = async () => {
    const response = await fetch(`${process.env.REACT_APP_API_URL}/analytics/dashboard`, {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    const data = await response.json();
    setAnalytics(data);
  };

  const checkUserRisk = async (user) => {
    const response = await fetch(`${process.env.REACT_APP_ML_URL}/predict-risk`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        user_id: user._id,
        login_frequency: user.loginFrequency || 5,
        failed_logins: user.failedLogins || 0,
        task_completion_rate: user.taskCompletionRate || 0.85,
        avg_response_time: user.avgResponseTime || 120
      })
    });
    const riskData = await response.json();
    alert(`Risk Level: ${riskData.risk_level}\nScore: ${riskData.risk_score}`);
  };

  return (
    <div className="admin-dashboard">
      <h1>Admin Dashboard</h1>
      
      {analytics && (
        <div className="analytics-cards">
          <div className="card">
            <h3>Total Users</h3>
            <p>{analytics.totalUsers}</p>
          </div>
          <div className="card">
            <h3>Active Tasks</h3>
            <p>{analytics.activeTasks}</p>
          </div>
          <div className="card">
            <h3>Open Tickets</h3>
            <p>{analytics.openTickets}</p>
          </div>
        </div>
      )}

      <div className="users-table">
        <h2>User Management</h2>
        <table>
          <thead>
            <tr>
              <th>Name</th>
              <th>Email</th>
              <th>Role</th>
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
                  <button onClick={() => checkUserRisk(user)}>Check Risk</button>
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

## Backend Implementation Patterns

### JWT Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');

const authMiddleware = (req, res, next) => {
  try {
    const token = req.headers.authorization?.split(' ')[1];
    
    if (!token) {
      return res.status(401).json({ message: 'No token provided' });
    }

    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch (error) {
    return res.status(401).json({ message: 'Invalid token' });
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

### User Model (MongoDB)

```javascript
// backend/models/User.js
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');

const userSchema = new mongoose.Schema({
  username: { type: String, required: true, unique: true },
  email: { type: String, required: true, unique: true },
  password: { type: String, required: true },
  role: { type: String, enum: ['user', 'admin'], default: 'user' },
  loginFrequency: { type: Number, default: 0 },
  failedLogins: { type: Number, default: 0 },
  taskCompletionRate: { type: Number, default: 0 },
  createdAt: { type: Date, default: Date.now },
  updatedAt: { type: Date, default: Date.now }
});

userSchema.pre('save', async function(next) {
  if (this.isModified('password')) {
    this.password = await bcrypt.hash(this.password, 10);
  }
  next();
});

userSchema.methods.comparePassword = async function(candidatePassword) {
  return bcrypt.compare(candidatePassword, this.password);
};

module.exports = mongoose.model('User', userSchema);
```

### Task Controller

```javascript
// backend/controllers/taskController.js
const Task = require('../models/Task');

exports.createTask = async (req, res) => {
  try {
    const task = new Task({
      ...req.body,
      createdBy: req.user.id
    });
    await task.save();
    res.status(201).json(task);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
};

exports.getUserTasks = async (req, res) => {
  try {
    const tasks = await Task.find({ assignedTo: req.params.userId })
      .populate('assignedTo', 'username email')
      .sort({ createdAt: -1 });
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
      { new: true }
    );
    res.json(task);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
};
```

## ML Service Implementation

### FastAPI Main Application

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import List, Optional
import joblib
import numpy as np
from sklearn.ensemble import RandomForestClassifier
from river import anomaly

app = FastAPI(title="Enterprise User Management ML Service")

# Pydantic models
class TicketClassificationRequest(BaseModel):
    text: str
    title: str

class RiskPredictionRequest(BaseModel):
    user_id: str
    login_frequency: int
    failed_logins: int
    task_completion_rate: float
    avg_response_time: float

class AnomalyDetectionRequest(BaseModel):
    user_id: str
    timestamp: str
    action_type: str
    resource_accessed: str
    ip_address: str

class BurnoutDetectionRequest(BaseModel):
    user_id: str
    tasks_assigned: int
    tasks_completed: int
    avg_hours_worked: float
    overtime_hours: float
    missed_deadlines: int

# Load or initialize models
try:
    risk_model = joblib.load('./models/risk_model.pkl')
except:
    risk_model = RandomForestClassifier(n_estimators=100)

anomaly_detector = anomaly.HalfSpaceTrees()

@app.post("/classify-ticket")
async def classify_ticket(request: TicketClassificationRequest):
    """Classify support ticket using NLP"""
    text_lower = request.text.lower()
    
    # Simple rule-based classification (replace with trained model)
    if any(word in text_lower for word in ['bug', 'error', 'crash', 'broken']):
        category = 'technical'
        priority = 'high'
    elif any(word in text_lower for word in ['feature', 'request', 'enhancement']):
        category = 'feature_request'
        priority = 'medium'
    elif any(word in text_lower for word in ['question', 'how to', 'help']):
        category = 'support'
        priority = 'low'
    else:
        category = 'general'
        priority = 'medium'
    
    return {
        "category": category,
        "priority": priority,
        "suggested_assignee": f"dept_{category}"
    }

@app.post("/predict-risk")
async def predict_risk(request: RiskPredictionRequest):
    """Predict user risk based on behavior patterns"""
    features = np.array([[
        request.login_frequency,
        request.failed_logins,
        request.task_completion_rate,
        request.avg_response_time
    ]])
    
    # Calculate risk score (0-1)
    risk_score = (
        (request.failed_logins * 0.3) +
        ((1 - request.task_completion_rate) * 0.4) +
        (min(request.avg_response_time / 600, 1) * 0.3)
    )
    
    if risk_score > 0.7:
        risk_level = 'high'
    elif risk_score > 0.4:
        risk_level = 'medium'
    else:
        risk_level = 'low'
    
    factors = []
    if request.failed_logins > 3:
        factors.append("High failed login attempts")
    if request.task_completion_rate < 0.6:
        factors.append("Low task completion rate")
    if request.avg_response_time > 300:
        factors.append("Slow response time")
    
    return {
        "risk_score": round(risk_score, 2),
        "risk_level": risk_level,
        "factors": factors
    }

@app.post("/detect-anomaly")
async def detect_anomaly(request: AnomalyDetectionRequest):
    """Detect anomalous user behavior"""
    # Create feature vector
    features = {
        'action_type': hash(request.action_type) % 100,
        'resource': hash(request.resource_accessed) % 100,
        'ip': hash(request.ip_address) % 100,
        'hour': int(request.timestamp.split('T')[1].split(':')[0]) if 'T' in request.timestamp else 12
    }
    
    # Update anomaly detector
    score = anomaly_detector.score_one(features)
    anomaly_detector.learn_one(features)
    
    is_anomaly = score > 0.7
    
    anomaly_type = None
    if is_anomaly:
        if features['hour'] < 6 or features['hour'] > 22:
            anomaly_type = 'unusual_time'
        else:
            anomaly_type = 'unusual_access'
    
    return {
        "is_anomaly": is_anomaly,
        "confidence": round(score, 2),
        "anomaly_type": anomaly_type
    }

@app.post("/detect-burnout")
async def detect_burnout(request: BurnoutDetectionRequest):
    """Detect employee burnout risk"""
    workload_ratio = request.tasks_assigned / max(request.tasks_completed, 1)
    overtime_factor = min(request.overtime_hours / 40, 1)
    deadline_miss_rate = request.missed_deadlines / max(request.tasks_assigned, 1)
    
    burnout_score = (
        (workload_ratio * 0.3) +
        (overtime_factor * 0.4) +
        (deadline_miss_rate * 0.3)
    )
    
    if burnout_score > 0.7:
        burnout_risk = 'high'
        recommendations = [
            "Reduce task assignments",
            "Schedule time off",
            "Review workload distribution"
        ]
    elif burnout_score > 0.4:
        burnout_risk = 'medium'
        recommendations = [
            "Monitor workload",
            "Encourage breaks"
        ]
    else:
        burnout_risk = 'low'
        recommendations = ["Continue current workload"]
    
    return {
        "burnout_risk": burnout_risk,
        "score": round(burnout_score, 2),
        "recommendations": recommendations
    }

@app.post("/predict-project-delay")
async def predict_project_delay(request: dict):
    """Predict likelihood of project delay"""
    completion_rate = request['completed_tasks'] / max(request['total_tasks'], 1)
    velocity = request['completed_tasks'] / max(request['team_size'], 1)
    
    expected_completion = (request['total_tasks'] - request['completed_tasks']) / max(velocity, 0.1)
    
    delay_probability = max(0, min(1, (expected_completion - request['days_remaining']) / request['days_remaining']))
    
    estimated_delay = max(0, expected_completion - request['days_remaining'])
    
    risk_factors = []
    if completion_rate < 0.5:
        risk_factors.append("Low completion rate")
    if velocity < 1:
        risk_factors.append("Low team velocity")
    if request['avg_task_completion_time'] > 5:
        risk_factors.append("High average task completion time")
    
    return {
        "delay_probability": round(delay_probability, 2),
        "estimated_delay_days": int(estimated_delay),
        "risk_factors": risk_factors
    }

@app.get("/health")
async def health_check():
    return {"status": "healthy"}
```

## Common Workflows

### Complete User Registration and Task Assignment Flow

```javascript
// Complete workflow: Register user, assign task, track progress
const completeUserOnboarding = async (newUserData) => {
  // Step 1: Register user
  const registerResponse = await fetch('http://localhost:5000/api/auth/register', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(newUserData)
  });
  const user = await registerResponse.json();
  
  // Step 2: Admin assigns task
  const adminToken = localStorage.getItem('adminToken');
  const taskResponse = await fetch('http://localhost:5000/api/tasks', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${adminToken}`
    },
    body: JSON.stringify({
      title: 'Onboarding Task',
      description: 'Complete user profile and training',
      assignedTo: user._id,
      priority: 'high',
      dueDate: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000)
    })
  });
  const task = await taskResponse.json();
  
  return { user, task };
};
```

### AI-Powered Ticket Routing

```javascript
// Automatically classify and route support ticket
const autoRouteTicket = async (ticketData, userToken) => {
  // Step 1: Create ticket
  const ticketResponse = await fetch('http://localhost:5000/api/tickets', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${userToken}`
    },
    body: JSON.stringify(ticketData)
  });
  const ticket = await ticketResponse.json();
  
  // Step 2: Classify with AI
  const classifyResponse = await fetch('http://localhost:8000/classify-ticket', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      text
