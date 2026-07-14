---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics for risk detection, burnout analysis, and intelligent task management
triggers:
  - "help me set up the enterprise user management system"
  - "how do I integrate AI analytics into user management"
  - "show me how to implement risk detection for users"
  - "configure the user management dashboard with AI insights"
  - "set up task tracking with burnout detection"
  - "implement ticket classification with machine learning"
  - "create a user management system with predictive analytics"
  - "build role-based access control with AI monitoring"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI coding agents to work with the Enterprise User Management System with AI Analytics, a full-stack application that combines user management, task tracking, support ticket handling, and AI-powered insights including risk detection, anomaly detection, burnout analysis, and predictive analytics.

## What This Project Does

The Enterprise User Management System provides:
- **User Management**: Role-based access control (Admin/User) with JWT authentication
- **Task Management**: Kanban board interface with time tracking
- **Support Tickets**: AI-powered ticket classification and routing
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, and project insights
- **Dashboards**: Separate admin and user dashboards with real-time metrics
- **Audit Logging**: Track user activities and suspicious behavior

## Project Architecture

The system consists of three main components:
1. **Frontend** (React.js): User interface and dashboards
2. **Backend** (Node.js): REST API, authentication, business logic
3. **ML Service** (FastAPI + Python): AI/ML models for analytics

## Installation

### Prerequisites
```bash
# Required software
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
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
JWT_SECRET=${JWT_SECRET}
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
EOF

npm start
```

### ML Service Setup
```bash
cd ml-service
pip install -r requirements.txt

# Create .env file
cat > .env << EOF
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
MODEL_PATH=./models
LOG_LEVEL=INFO
EOF

uvicorn main:app --reload --port 8000
```

### Frontend Setup
```bash
cd frontend
npm install

# Create .env file
cat > .env << EOF
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_SERVICE_URL=http://localhost:8000
EOF

npm start
```

## Key API Endpoints

### Authentication
```javascript
// POST /api/auth/register
const registerUser = async (userData) => {
  const response = await fetch('http://localhost:5000/api/auth/register', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      name: userData.name,
      email: userData.email,
      password: userData.password,
      role: userData.role // 'admin' or 'user'
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
      priority: taskData.priority, // 'low', 'medium', 'high'
      dueDate: taskData.dueDate,
      status: 'todo' // 'todo', 'in-progress', 'done'
    })
  });
  return response.json();
};

// PUT /api/tasks/:id/status - Update task status
const updateTaskStatus = async (taskId, newStatus, token) => {
  const response = await fetch(`http://localhost:5000/api/tasks/${taskId}/status`, {
    method: 'PUT',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify({ status: newStatus })
  });
  return response.json();
};

// POST /api/tasks/:id/time - Track time on task
const trackTime = async (taskId, timeData, token) => {
  const response = await fetch(`http://localhost:5000/api/tasks/${taskId}/time`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify({
      startTime: timeData.startTime,
      endTime: timeData.endTime,
      duration: timeData.duration
    })
  });
  return response.json();
};
```

### Support Tickets
```javascript
// POST /api/tickets - Create support ticket
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

// GET /api/tickets/:id - Get ticket with AI classification
const getTicket = async (ticketId, token) => {
  const response = await fetch(`http://localhost:5000/api/tickets/${ticketId}`, {
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return response.json();
};
```

## AI/ML Service Integration

### Risk Detection
```javascript
// POST /api/ml/risk-detection - Analyze user risk
const analyzeUserRisk = async (userId, token) => {
  const response = await fetch('http://localhost:5000/api/ml/risk-detection', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify({
      userId: userId,
      metrics: {
        loginFrequency: 45,
        taskCompletionRate: 0.65,
        avgResponseTime: 3.2,
        missedDeadlines: 5,
        lateLogins: 8
      }
    })
  });
  const result = await response.json();
  // result: { riskScore: 0.72, riskLevel: 'high', factors: [...] }
  return result;
};
```

### Anomaly Detection
```javascript
// POST /api/ml/anomaly-detection - Detect unusual behavior
const detectAnomalies = async (activityData, token) => {
  const response = await fetch('http://localhost:5000/api/ml/anomaly-detection', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify({
      userId: activityData.userId,
      timestamp: new Date().toISOString(),
      activityType: activityData.type,
      features: {
        loginTime: activityData.loginTime,
        location: activityData.location,
        deviceType: activityData.device,
        actionCount: activityData.actions
      }
    })
  });
  const result = await response.json();
  // result: { isAnomaly: true, anomalyScore: 0.89, reason: '...' }
  return result;
};
```

### Burnout Analysis
```javascript
// POST /api/ml/burnout-detection - Analyze employee burnout risk
const analyzeBurnout = async (userId, token) => {
  const response = await fetch('http://localhost:5000/api/ml/burnout-detection', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify({
      userId: userId,
      period: '30days',
      metrics: {
        hoursWorked: 220,
        overtimeHours: 45,
        tasksCompleted: 28,
        avgTaskDuration: 7.5,
        weekendWork: 6,
        stressLevel: 7
      }
    })
  });
  const result = await response.json();
  // result: { burnoutRisk: 'high', score: 0.78, recommendations: [...] }
  return result;
};
```

### Ticket Classification
```javascript
// POST /api/ml/classify-ticket - AI-powered ticket routing
const classifyTicket = async (ticketData, token) => {
  const response = await fetch('http://localhost:5000/api/ml/classify-ticket', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify({
      title: ticketData.title,
      description: ticketData.description
    })
  });
  const result = await response.json();
  // result: { category: 'technical', priority: 'high', department: 'IT' }
  return result;
};
```

### Predictive Project Insights
```javascript
// POST /api/ml/project-insights - Predict project outcomes
const getProjectInsights = async (projectData, token) => {
  const response = await fetch('http://localhost:5000/api/ml/project-insights', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify({
      projectId: projectData.id,
      completedTasks: projectData.completed,
      totalTasks: projectData.total,
      teamSize: projectData.teamSize,
      deadline: projectData.deadline,
      currentVelocity: projectData.velocity
    })
  });
  const result = await response.json();
  // result: { onTrack: false, delayPrediction: 5, confidence: 0.85 }
  return result;
};
```

## React Component Patterns

### Admin Dashboard Component
```javascript
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const AdminDashboard = () => {
  const [users, setUsers] = useState([]);
  const [analytics, setAnalytics] = useState(null);
  const [loading, setLoading] = useState(true);
  
  const token = localStorage.getItem('token');
  
  useEffect(() => {
    fetchDashboardData();
  }, []);
  
  const fetchDashboardData = async () => {
    try {
      const config = {
        headers: { Authorization: `Bearer ${token}` }
      };
      
      const [usersRes, analyticsRes] = await Promise.all([
        axios.get('http://localhost:5000/api/users', config),
        axios.get('http://localhost:5000/api/analytics/overview', config)
      ]);
      
      setUsers(usersRes.data);
      setAnalytics(analyticsRes.data);
      setLoading(false);
    } catch (error) {
      console.error('Error fetching dashboard data:', error);
    }
  };
  
  const checkUserRisk = async (userId) => {
    try {
      const response = await axios.post(
        'http://localhost:5000/api/ml/risk-detection',
        { userId },
        { headers: { Authorization: `Bearer ${token}` } }
      );
      
      alert(`Risk Level: ${response.data.riskLevel} (Score: ${response.data.riskScore})`);
    } catch (error) {
      console.error('Error checking risk:', error);
    }
  };
  
  if (loading) return <div>Loading dashboard...</div>;
  
  return (
    <div className="admin-dashboard">
      <h1>Admin Dashboard</h1>
      
      <div className="metrics">
        <div className="metric-card">
          <h3>Total Users</h3>
          <p>{analytics.totalUsers}</p>
        </div>
        <div className="metric-card">
          <h3>Active Tasks</h3>
          <p>{analytics.activeTasks}</p>
        </div>
        <div className="metric-card">
          <h3>Open Tickets</h3>
          <p>{analytics.openTickets}</p>
        </div>
      </div>
      
      <div className="users-table">
        <h2>User Management</h2>
        <table>
          <thead>
            <tr>
              <th>Name</th>
              <th>Email</th>
              <th>Role</th>
              <th>Status</th>
              <th>Actions</th>
            </tr>
          </thead>
          <tbody>
            {users.map(user => (
              <tr key={user._id}>
                <td>{user.name}</td>
                <td>{user.email}</td>
                <td>{user.role}</td>
                <td>{user.status}</td>
                <td>
                  <button onClick={() => checkUserRisk(user._id)}>
                    Check Risk
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

export default AdminDashboard;
```

### Kanban Board Component
```javascript
import React, { useState, useEffect } from 'react';
import { DragDropContext, Droppable, Draggable } from 'react-beautiful-dnd';
import axios from 'axios';

const KanbanBoard = () => {
  const [tasks, setTasks] = useState({
    todo: [],
    'in-progress': [],
    done: []
  });
  
  const token = localStorage.getItem('token');
  
  useEffect(() => {
    fetchTasks();
  }, []);
  
  const fetchTasks = async () => {
    try {
      const response = await axios.get('http://localhost:5000/api/tasks', {
        headers: { Authorization: `Bearer ${token}` }
      });
      
      const grouped = response.data.reduce((acc, task) => {
        acc[task.status].push(task);
        return acc;
      }, { todo: [], 'in-progress': [], done: [] });
      
      setTasks(grouped);
    } catch (error) {
      console.error('Error fetching tasks:', error);
    }
  };
  
  const onDragEnd = async (result) => {
    if (!result.destination) return;
    
    const { source, destination, draggableId } = result;
    
    if (source.droppableId === destination.droppableId) return;
    
    try {
      await axios.put(
        `http://localhost:5000/api/tasks/${draggableId}/status`,
        { status: destination.droppableId },
        { headers: { Authorization: `Bearer ${token}` } }
      );
      
      fetchTasks();
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };
  
  return (
    <DragDropContext onDragEnd={onDragEnd}>
      <div className="kanban-board">
        {['todo', 'in-progress', 'done'].map(status => (
          <Droppable key={status} droppableId={status}>
            {(provided) => (
              <div
                className="kanban-column"
                ref={provided.innerRef}
                {...provided.droppableProps}
              >
                <h3>{status.toUpperCase()}</h3>
                {tasks[status].map((task, index) => (
                  <Draggable
                    key={task._id}
                    draggableId={task._id}
                    index={index}
                  >
                    {(provided) => (
                      <div
                        className="task-card"
                        ref={provided.innerRef}
                        {...provided.draggableProps}
                        {...provided.dragHandleProps}
                      >
                        <h4>{task.title}</h4>
                        <p>{task.description}</p>
                        <span className={`priority ${task.priority}`}>
                          {task.priority}
                        </span>
                      </div>
                    )}
                  </Draggable>
                ))}
                {provided.placeholder}
              </div>
            )}
          </Droppable>
        ))}
      </div>
    </DragDropContext>
  );
};

export default KanbanBoard;
```

## Backend Node.js Patterns

### JWT Authentication Middleware
```javascript
const jwt = require('jsonwebtoken');

const authMiddleware = async (req, res, next) => {
  try {
    const token = req.header('Authorization')?.replace('Bearer ', '');
    
    if (!token) {
      return res.status(401).json({ error: 'Authentication required' });
    }
    
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.userId = decoded.id;
    req.userRole = decoded.role;
    
    next();
  } catch (error) {
    res.status(401).json({ error: 'Invalid token' });
  }
};

const adminOnly = (req, res, next) => {
  if (req.userRole !== 'admin') {
    return res.status(403).json({ error: 'Admin access required' });
  }
  next();
};

module.exports = { authMiddleware, adminOnly };
```

### User Routes
```javascript
const express = require('express');
const router = express.Router();
const User = require('../models/User');
const { authMiddleware, adminOnly } = require('../middleware/auth');

// Get all users (admin only)
router.get('/', authMiddleware, adminOnly, async (req, res) => {
  try {
    const users = await User.find().select('-password');
    res.json(users);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Get user by ID
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
router.put('/:id', authMiddleware, adminOnly, async (req, res) => {
  try {
    const { name, email, role, status } = req.body;
    const user = await User.findByIdAndUpdate(
      req.params.id,
      { name, email, role, status },
      { new: true, runValidators: true }
    ).select('-password');
    
    res.json(user);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Delete user
router.delete('/:id', authMiddleware, adminOnly, async (req, res) => {
  try {
    await User.findByIdAndDelete(req.params.id);
    res.json({ message: 'User deleted successfully' });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

module.exports = router;
```

### Task Controller
```javascript
const Task = require('../models/Task');
const axios = require('axios');

exports.createTask = async (req, res) => {
  try {
    const { title, description, assignedTo, priority, dueDate } = req.body;
    
    const task = new Task({
      title,
      description,
      assignedTo,
      priority,
      dueDate,
      createdBy: req.userId,
      status: 'todo'
    });
    
    await task.save();
    
    // Trigger AI analysis
    try {
      await axios.post(`${process.env.ML_SERVICE_URL}/analyze-task`, {
        taskId: task._id,
        assignedTo
      });
    } catch (mlError) {
      console.error('ML service error:', mlError.message);
    }
    
    res.status(201).json(task);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
};

exports.updateTaskStatus = async (req, res) => {
  try {
    const { status } = req.body;
    const task = await Task.findByIdAndUpdate(
      req.params.id,
      { status, updatedAt: new Date() },
      { new: true }
    );
    
    if (!task) {
      return res.status(404).json({ error: 'Task not found' });
    }
    
    res.json(task);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
};

exports.trackTime = async (req, res) => {
  try {
    const { startTime, endTime, duration } = req.body;
    const task = await Task.findById(req.params.id);
    
    if (!task) {
      return res.status(404).json({ error: 'Task not found' });
    }
    
    task.timeEntries.push({
      userId: req.userId,
      startTime,
      endTime,
      duration
    });
    
    await task.save();
    res.json(task);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
};

module.exports = exports;
```

## Python ML Service Patterns

### FastAPI Main Application
```python
from fastapi import FastAPI, HTTPException, Depends
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from typing import List, Optional
import joblib
import numpy as np
from sklearn.ensemble import RandomForestClassifier
from river import anomaly
import os

app = FastAPI(title="Enterprise User Management ML Service")

# CORS configuration
app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:3000", "http://localhost:5000"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Load or initialize models
MODEL_PATH = os.getenv('MODEL_PATH', './models')

class RiskDetectionRequest(BaseModel):
    userId: str
    metrics: dict

class RiskDetectionResponse(BaseModel):
    riskScore: float
    riskLevel: str
    factors: List[str]

@app.post("/api/ml/risk-detection", response_model=RiskDetectionResponse)
async def detect_risk(request: RiskDetectionRequest):
    try:
        # Extract features
        features = [
            request.metrics.get('loginFrequency', 0),
            request.metrics.get('taskCompletionRate', 0),
            request.metrics.get('avgResponseTime', 0),
            request.metrics.get('missedDeadlines', 0),
            request.metrics.get('lateLogins', 0)
        ]
        
        # Calculate risk score (simplified)
        risk_score = calculate_risk_score(features)
        
        # Determine risk level
        if risk_score > 0.7:
            risk_level = "high"
        elif risk_score > 0.4:
            risk_level = "medium"
        else:
            risk_level = "low"
        
        # Identify risk factors
        factors = identify_risk_factors(request.metrics)
        
        return RiskDetectionResponse(
            riskScore=risk_score,
            riskLevel=risk_level,
            factors=factors
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

def calculate_risk_score(features):
    # Weighted scoring algorithm
    weights = [0.15, 0.30, 0.20, 0.25, 0.10]
    normalized = normalize_features(features)
    score = sum(w * f for w, f in zip(weights, normalized))
    return min(max(score, 0.0), 1.0)

def normalize_features(features):
    # Normalize to 0-1 range
    max_values = [100, 1.0, 10.0, 20, 30]
    return [min(f / m, 1.0) for f, m in zip(features, max_values)]

def identify_risk_factors(metrics):
    factors = []
    if metrics.get('taskCompletionRate', 1.0) < 0.7:
        factors.append("Low task completion rate")
    if metrics.get('missedDeadlines', 0) > 5:
        factors.append("Frequent missed deadlines")
    if metrics.get('lateLogins', 0) > 10:
        factors.append("Irregular login patterns")
    if metrics.get('avgResponseTime', 0) > 5:
        factors.append("Slow response times")
    return factors
```

### Anomaly Detection Service
```python
from river import anomaly
from datetime import datetime

# Initialize anomaly detector
anomaly_detector = anomaly.HalfSpaceTrees(n_trees=10, height=8, window_size=250)

class AnomalyDetectionRequest(BaseModel):
    userId: str
    timestamp: str
    activityType: str
    features: dict

class AnomalyDetectionResponse(BaseModel):
    isAnomaly: bool
    anomalyScore: float
    reason: Optional[str]

@app.post("/api/ml/anomaly-detection", response_model=AnomalyDetectionResponse)
async def detect_anomaly(request: AnomalyDetectionRequest):
    try:
        # Prepare feature vector
        feature_vector = {
            'hour': datetime.fromisoformat(request.timestamp.replace('Z', '+00:00')).hour,
            'login_time': request.features.get('loginTime', 0),
            'action_count': request.features.get('actionCount', 0),
            'device_type': hash(request.features.get('deviceType', '')) % 100
        }
        
        # Get anomaly score
        score = anomaly_detector.score_one(feature_vector)
        anomaly_detector.learn_one(feature_vector)
        
        # Determine if anomalous (threshold = 0.8)
        is_anomaly = score > 0.8
        
        reason = None
        if is_anomaly:
            reason = determine_anomaly_reason(request.features, score)
        
        return AnomalyDetectionResponse(
            isAnomaly=is_anomaly,
            anomalyScore=score,
            reason=reason
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

def determine_anomaly_reason(features, score):
    reasons = []
    
    if features.get('loginTime', 0) < 6 or features.get('loginTime', 0) > 22:
        reasons.append("Unusual login time")
    
    if features.get('actionCount', 0) > 100:
        reasons.append("Unusually high activity")
    
    if not reasons:
        reasons.append(f"Anomalous pattern detected (score: {score:.2f})")
    
    return "; ".join(reasons)
```

### Burnout Detection Service
```python
class BurnoutDetectionRequest(BaseModel):
    userId: str
    period: str
    metrics: dict

class BurnoutDetectionResponse(BaseModel):
    burnoutRisk: str
    score: float
    recommendations: List[str]

@app.post("/api/ml/burnout-detection", response_model=BurnoutDetectionResponse)
async def detect_burnout(request: BurnoutDetectionRequest):
    try:
        metrics = request.metrics
        
        # Calculate burnout indicators
        overtime_ratio = metrics.get('overtimeHours', 0) / max(metrics.get('hoursWorked', 1), 1)
        weekend_work_ratio = metrics.get('weekendWork', 0) / 4  # Assuming monthly period
        stress_normalized = metrics.get('stressLevel', 0) / 10
        task_burden = metrics.get('avgTaskDuration', 0) / 10
        
        # Weighted burnout score
        burnout_score = (
            overtime_ratio * 0.3 +
            weekend_work_ratio * 0.25 +
            stress_normalized * 0.3 +
            task_burden * 0.15
        )
        
        # Determine risk level
        if burnout_score > 0.
