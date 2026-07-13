---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics, risk detection, and task automation using React, Node.js, and FastAPI ML service
triggers:
  - how do I set up the enterprise user management system
  - integrate AI analytics into user management
  - implement JWT authentication for user roles
  - create AI-powered ticket classification system
  - build admin dashboard with user analytics
  - add burnout detection and anomaly alerts
  - setup ML service for risk prediction
  - configure Kanban board with task tracking
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System is a full-stack application that combines traditional user/task management with AI-powered analytics. It provides role-based access control, task tracking with Kanban boards, support ticket management, and ML-driven insights including risk prediction, anomaly detection, burnout analysis, and predictive project analytics.

**Architecture:**
- **Frontend:** React.js (port 3000)
- **Backend:** Node.js/Express REST API (port 5000)
- **ML Service:** FastAPI with scikit-learn and River (port 8000)
- **Database:** MongoDB
- **Auth:** JWT-based authentication

## Installation

### Prerequisites

```bash
# Required
node >= 14.x
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
MONGODB_URI=mongodb://localhost:27017/enterprise_ums
JWT_SECRET=${JWT_SECRET}
JWT_EXPIRE=7d
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
MONGODB_URI=mongodb://localhost:27017/enterprise_ums
MODEL_PATH=./models
LOG_LEVEL=INFO
EOF

# Start ML service
uvicorn main:app --reload --port 8000
```

### Frontend Setup

```bash
cd frontend
npm install

# Create .env file
cat > .env << EOF
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_API_URL=http://localhost:8000
EOF

# Start frontend
npm start
```

## Key API Endpoints

### Authentication

```javascript
// POST /api/auth/register
{
  "username": "john_doe",
  "email": "john@example.com",
  "password": "SecurePass123",
  "role": "user" // or "admin"
}

// POST /api/auth/login
{
  "email": "john@example.com",
  "password": "SecurePass123"
}

// Response includes JWT token
{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "user": {
    "id": "507f1f77bcf86cd799439011",
    "username": "john_doe",
    "role": "user"
  }
}
```

### User Management (Admin)

```javascript
// GET /api/users - List all users
// Headers: Authorization: Bearer {token}

// PUT /api/users/:userId
{
  "role": "admin",
  "status": "active",
  "department": "Engineering"
}

// DELETE /api/users/:userId
```

### Task Management

```javascript
// POST /api/tasks
{
  "title": "Implement authentication",
  "description": "Add JWT-based auth",
  "assignedTo": "507f1f77bcf86cd799439011",
  "priority": "high",
  "status": "todo", // "todo", "in-progress", "done"
  "dueDate": "2026-05-01"
}

// PATCH /api/tasks/:taskId/status
{
  "status": "in-progress"
}

// POST /api/tasks/:taskId/time
{
  "action": "start" // or "stop"
}
```

### Support Tickets

```javascript
// POST /api/tickets
{
  "title": "Cannot access dashboard",
  "description": "Getting 403 error",
  "priority": "high",
  "category": "technical"
}

// GET /api/tickets - List tickets (filtered by user role)
```

### AI Analytics Endpoints

```python
# POST /ml/classify-ticket
{
  "title": "Cannot login to system",
  "description": "Getting authentication failed error"
}
# Response: {"category": "technical", "priority": "high", "confidence": 0.87}

# POST /ml/detect-risk
{
  "userId": "507f1f77bcf86cd799439011",
  "activityData": {
    "failedLogins": 5,
    "dataAccessPatterns": [...],
    "workingHours": [...]
  }
}
# Response: {"riskScore": 0.73, "riskLevel": "high", "factors": [...]}

# POST /ml/detect-burnout
{
  "userId": "507f1f77bcf86cd799439011",
  "workload": {
    "tasksCompleted": 45,
    "averageWorkHours": 11.2,
    "overtimeHours": 25,
    "missedDeadlines": 3
  }
}
# Response: {"burnoutRisk": 0.82, "recommendation": "reduce workload"}

# POST /ml/predict-delay
{
  "projectId": "507f1f77bcf86cd799439011",
  "tasks": [...],
  "timeline": {...}
}
# Response: {"delayProbability": 0.65, "estimatedDelay": 5}
```

## Frontend Components

### User Dashboard Implementation

```javascript
// src/pages/UserDashboard.js
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const UserDashboard = () => {
  const [tasks, setTasks] = useState([]);
  const [tickets, setTickets] = useState([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchUserData();
  }, []);

  const fetchUserData = async () => {
    try {
      const token = localStorage.getItem('token');
      const config = {
        headers: { Authorization: `Bearer ${token}` }
      };

      const [tasksRes, ticketsRes] = await Promise.all([
        axios.get(`${process.env.REACT_APP_API_URL}/tasks/my-tasks`, config),
        axios.get(`${process.env.REACT_APP_API_URL}/tickets/my-tickets`, config)
      ]);

      setTasks(tasksRes.data);
      setTickets(ticketsRes.data);
      setLoading(false);
    } catch (error) {
      console.error('Error fetching data:', error);
    }
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    try {
      const token = localStorage.getItem('token');
      await axios.patch(
        `${process.env.REACT_APP_API_URL}/tasks/${taskId}/status`,
        { status: newStatus },
        { headers: { Authorization: `Bearer ${token}` } }
      );
      fetchUserData();
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  return (
    <div className="dashboard">
      <h1>My Dashboard</h1>
      
      <div className="stats">
        <div className="stat-card">
          <h3>Active Tasks</h3>
          <p>{tasks.filter(t => t.status !== 'done').length}</p>
        </div>
        <div className="stat-card">
          <h3>Open Tickets</h3>
          <p>{tickets.filter(t => t.status === 'open').length}</p>
        </div>
      </div>

      <div className="kanban-board">
        {['todo', 'in-progress', 'done'].map(status => (
          <div key={status} className="kanban-column">
            <h3>{status.toUpperCase()}</h3>
            {tasks.filter(t => t.status === status).map(task => (
              <div key={task._id} className="task-card">
                <h4>{task.title}</h4>
                <p>{task.description}</p>
                <select 
                  value={task.status}
                  onChange={(e) => updateTaskStatus(task._id, e.target.value)}
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
    </div>
  );
};

export default UserDashboard;
```

### Admin Analytics Dashboard

```javascript
// src/pages/AdminDashboard.js
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const AdminDashboard = () => {
  const [analytics, setAnalytics] = useState(null);
  const [users, setUsers] = useState([]);
  const [alerts, setAlerts] = useState([]);

  useEffect(() => {
    fetchAdminData();
  }, []);

  const fetchAdminData = async () => {
    try {
      const token = localStorage.getItem('token');
      const config = {
        headers: { Authorization: `Bearer ${token}` }
      };

      const [analyticsRes, usersRes, alertsRes] = await Promise.all([
        axios.get(`${process.env.REACT_APP_API_URL}/admin/analytics`, config),
        axios.get(`${process.env.REACT_APP_API_URL}/users`, config),
        axios.get(`${process.env.REACT_APP_ML_API_URL}/ml/alerts`, config)
      ]);

      setAnalytics(analyticsRes.data);
      setUsers(usersRes.data);
      setAlerts(alertsRes.data);
    } catch (error) {
      console.error('Error fetching admin data:', error);
    }
  };

  const checkUserRisk = async (userId) => {
    try {
      const token = localStorage.getItem('token');
      const response = await axios.post(
        `${process.env.REACT_APP_ML_API_URL}/ml/detect-risk`,
        { userId },
        { headers: { Authorization: `Bearer ${token}` } }
      );
      
      alert(`Risk Score: ${response.data.riskScore}`);
    } catch (error) {
      console.error('Error checking risk:', error);
    }
  };

  return (
    <div className="admin-dashboard">
      <h1>Admin Dashboard</h1>

      <div className="analytics-section">
        <div className="metric">
          <h3>Total Users</h3>
          <p>{analytics?.totalUsers || 0}</p>
        </div>
        <div className="metric">
          <h3>Active Tasks</h3>
          <p>{analytics?.activeTasks || 0}</p>
        </div>
        <div className="metric">
          <h3>Open Tickets</h3>
          <p>{analytics?.openTickets || 0}</p>
        </div>
      </div>

      <div className="alerts-section">
        <h2>AI Alerts</h2>
        {alerts.map(alert => (
          <div key={alert._id} className={`alert alert-${alert.severity}`}>
            <h4>{alert.type}</h4>
            <p>{alert.message}</p>
            <span>{new Date(alert.timestamp).toLocaleString()}</span>
          </div>
        ))}
      </div>

      <div className="users-section">
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

## Backend Implementation

### Authentication Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

const protect = async (req, res, next) => {
  let token;

  if (req.headers.authorization && 
      req.headers.authorization.startsWith('Bearer')) {
    token = req.headers.authorization.split(' ')[1];
  }

  if (!token) {
    return res.status(401).json({ 
      message: 'Not authorized to access this route' 
    });
  }

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = await User.findById(decoded.id).select('-password');
    next();
  } catch (error) {
    return res.status(401).json({ 
      message: 'Not authorized to access this route' 
    });
  }
};

const authorize = (...roles) => {
  return (req, res, next) => {
    if (!roles.includes(req.user.role)) {
      return res.status(403).json({
        message: `User role ${req.user.role} is not authorized`
      });
    }
    next();
  };
};

module.exports = { protect, authorize };
```

### Task Controller

```javascript
// backend/controllers/taskController.js
const Task = require('../models/Task');
const axios = require('axios');

// Create task
exports.createTask = async (req, res) => {
  try {
    const task = await Task.create({
      ...req.body,
      createdBy: req.user.id
    });

    res.status(201).json({
      success: true,
      data: task
    });
  } catch (error) {
    res.status(400).json({
      success: false,
      error: error.message
    });
  }
};

// Get user tasks
exports.getMyTasks = async (req, res) => {
  try {
    const tasks = await Task.find({
      $or: [
        { assignedTo: req.user.id },
        { createdBy: req.user.id }
      ]
    }).sort({ createdAt: -1 });

    res.status(200).json({
      success: true,
      count: tasks.length,
      data: tasks
    });
  } catch (error) {
    res.status(500).json({
      success: false,
      error: error.message
    });
  }
};

// Update task status
exports.updateTaskStatus = async (req, res) => {
  try {
    const { status } = req.body;
    const task = await Task.findById(req.params.id);

    if (!task) {
      return res.status(404).json({
        success: false,
        error: 'Task not found'
      });
    }

    // Check authorization
    if (task.assignedTo.toString() !== req.user.id && 
        req.user.role !== 'admin') {
      return res.status(403).json({
        success: false,
        error: 'Not authorized to update this task'
      });
    }

    task.status = status;
    if (status === 'done') {
      task.completedAt = Date.now();
    }
    await task.save();

    res.status(200).json({
      success: true,
      data: task
    });
  } catch (error) {
    res.status(500).json({
      success: false,
      error: error.message
    });
  }
};

// Track time
exports.trackTime = async (req, res) => {
  try {
    const { action } = req.body; // 'start' or 'stop'
    const task = await Task.findById(req.params.id);

    if (!task) {
      return res.status(404).json({
        success: false,
        error: 'Task not found'
      });
    }

    if (action === 'start') {
      task.timeTracking.push({
        startTime: Date.now(),
        userId: req.user.id
      });
    } else if (action === 'stop') {
      const lastEntry = task.timeTracking[task.timeTracking.length - 1];
      if (lastEntry && !lastEntry.endTime) {
        lastEntry.endTime = Date.now();
      }
    }

    await task.save();

    res.status(200).json({
      success: true,
      data: task
    });
  } catch (error) {
    res.status(500).json({
      success: false,
      error: error.message
    });
  }
};
```

### Ticket Controller with AI Integration

```javascript
// backend/controllers/ticketController.js
const Ticket = require('../models/Ticket');
const axios = require('axios');

// Create ticket with AI classification
exports.createTicket = async (req, res) => {
  try {
    const { title, description, priority, category } = req.body;

    // Use AI to classify ticket if not provided
    let aiClassification = {};
    if (!priority || !category) {
      try {
        const mlResponse = await axios.post(
          `${process.env.ML_SERVICE_URL}/ml/classify-ticket`,
          { title, description }
        );
        aiClassification = mlResponse.data;
      } catch (mlError) {
        console.error('ML service error:', mlError.message);
      }
    }

    const ticket = await Ticket.create({
      title,
      description,
      priority: priority || aiClassification.priority || 'medium',
      category: category || aiClassification.category || 'general',
      createdBy: req.user.id,
      aiClassified: !!aiClassification.category
    });

    res.status(201).json({
      success: true,
      data: ticket,
      aiSuggestions: aiClassification
    });
  } catch (error) {
    res.status(400).json({
      success: false,
      error: error.message
    });
  }
};

// Get tickets
exports.getTickets = async (req, res) => {
  try {
    const query = req.user.role === 'admin' 
      ? {} 
      : { createdBy: req.user.id };

    const tickets = await Ticket.find(query)
      .populate('createdBy', 'username email')
      .sort({ createdAt: -1 });

    res.status(200).json({
      success: true,
      count: tickets.length,
      data: tickets
    });
  } catch (error) {
    res.status(500).json({
      success: false,
      error: error.message
    });
  }
};
```

## ML Service Implementation

### FastAPI Main Application

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException, Depends
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from typing import List, Optional
import joblib
import numpy as np
from datetime import datetime
import os

app = FastAPI(title="Enterprise UMS ML Service")

# CORS configuration
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Models (load or initialize)
MODEL_PATH = os.getenv('MODEL_PATH', './models')

class TicketClassificationRequest(BaseModel):
    title: str
    description: str

class RiskDetectionRequest(BaseModel):
    userId: str
    activityData: dict

class BurnoutDetectionRequest(BaseModel):
    userId: str
    workload: dict

class ProjectDelayRequest(BaseModel):
    projectId: str
    tasks: List[dict]
    timeline: dict

@app.post("/ml/classify-ticket")
async def classify_ticket(request: TicketClassificationRequest):
    """
    Classify ticket category and priority using NLP
    """
    try:
        # Simple keyword-based classification (replace with trained model)
        text = f"{request.title} {request.description}".lower()
        
        # Category classification
        if any(word in text for word in ['login', 'password', 'access', 'auth']):
            category = 'technical'
        elif any(word in text for word in ['payment', 'invoice', 'billing']):
            category = 'billing'
        elif any(word in text for word in ['slow', 'performance', 'crash']):
            category = 'technical'
        else:
            category = 'general'
        
        # Priority classification
        if any(word in text for word in ['urgent', 'critical', 'down', 'broken']):
            priority = 'high'
        elif any(word in text for word in ['soon', 'important']):
            priority = 'medium'
        else:
            priority = 'low'
        
        confidence = 0.85  # Mock confidence score
        
        return {
            "category": category,
            "priority": priority,
            "confidence": confidence,
            "timestamp": datetime.utcnow().isoformat()
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/ml/detect-risk")
async def detect_risk(request: RiskDetectionRequest):
    """
    Detect user risk based on activity patterns
    """
    try:
        activity = request.activityData
        
        # Calculate risk score based on various factors
        risk_factors = []
        risk_score = 0.0
        
        # Failed logins
        failed_logins = activity.get('failedLogins', 0)
        if failed_logins > 3:
            risk_score += 0.3
            risk_factors.append(f"High failed login attempts: {failed_logins}")
        
        # Unusual access patterns
        unusual_access = activity.get('unusualAccessPatterns', False)
        if unusual_access:
            risk_score += 0.25
            risk_factors.append("Unusual data access patterns detected")
        
        # Working hours anomaly
        avg_hours = activity.get('averageWorkHours', 8)
        if avg_hours > 12:
            risk_score += 0.2
            risk_factors.append(f"Excessive working hours: {avg_hours}")
        
        # Data download volume
        data_volume = activity.get('dataDownloadVolume', 0)
        if data_volume > 1000:  # MB
            risk_score += 0.25
            risk_factors.append(f"High data download: {data_volume}MB")
        
        risk_score = min(risk_score, 1.0)
        
        if risk_score > 0.7:
            risk_level = 'high'
        elif risk_score > 0.4:
            risk_level = 'medium'
        else:
            risk_level = 'low'
        
        return {
            "userId": request.userId,
            "riskScore": round(risk_score, 2),
            "riskLevel": risk_level,
            "factors": risk_factors,
            "timestamp": datetime.utcnow().isoformat()
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/ml/detect-burnout")
async def detect_burnout(request: BurnoutDetectionRequest):
    """
    Detect employee burnout risk based on workload metrics
    """
    try:
        workload = request.workload
        
        burnout_score = 0.0
        factors = []
        
        # Tasks completed (too many or too few)
        tasks = workload.get('tasksCompleted', 0)
        if tasks > 50:
            burnout_score += 0.25
            factors.append(f"High task volume: {tasks} tasks")
        
        # Average work hours
        avg_hours = workload.get('averageWorkHours', 8)
        if avg_hours > 10:
            burnout_score += 0.3
            factors.append(f"Excessive hours: {avg_hours} hrs/day")
        
        # Overtime hours
        overtime = workload.get('overtimeHours', 0)
        if overtime > 20:
            burnout_score += 0.25
            factors.append(f"High overtime: {overtime} hours")
        
        # Missed deadlines
        missed = workload.get('missedDeadlines', 0)
        if missed > 2:
            burnout_score += 0.2
            factors.append(f"Missed deadlines: {missed}")
        
        burnout_score = min(burnout_score, 1.0)
        
        # Recommendations
        recommendations = []
        if burnout_score > 0.6:
            recommendations.extend([
                "Reduce workload immediately",
                "Consider time off",
                "Reassign non-critical tasks"
            ])
        elif burnout_score > 0.4:
            recommendations.extend([
                "Monitor workload closely",
                "Encourage regular breaks",
                "Review task priorities"
            ])
        
        return {
            "userId": request.userId,
            "burnoutRisk": round(burnout_score, 2),
            "riskLevel": "high" if burnout_score > 0.6 else "medium" if burnout_score > 0.3 else "low",
            "factors": factors,
            "recommendations": recommendations,
            "timestamp": datetime.utcnow().isoformat()
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/ml/predict-delay")
async def predict_delay(request: ProjectDelayRequest):
    """
    Predict project delay probability
    """
    try:
        tasks = request.tasks
        timeline = request.timeline
        
        delay_score = 0.0
        factors = []
        
        # Calculate completion rate
        total_tasks = len(tasks)
        completed = sum(1 for t in tasks if t.get('status') == 'done')
        completion_rate = completed / total_tasks if total_tasks > 0 else 0
        
        # Check if behind schedule
        expected_completion = timeline.get('expectedCompletion', 0.5)
        if completion_rate < expected_completion:
            delay_score += 0.4
            factors.append(f"Behind schedule: {completion_rate*100:.1f}% vs {expected_completion*100:.1f}%")
        
        # High priority tasks pending
        high_priority_pending = sum(
            1 for t in tasks 
            if t.get('priority') == 'high' and t.get('status') != 'done'
        )
        if high_priority_pending > 3:
            delay_score += 0.3
            factors.append(f"High priority tasks pending: {high_priority_pending}")
        
        # Resource availability
        resource_util = timeline.get('resourceUtilization', 0.7)
        if resource_util > 0.9:
            delay_score += 0.3
            factors.append(f"High resource utilization: {resource_util*100:.1f}%")
        
        delay_score = min(delay_score, 1.0)
        
        # Estimate delay in days
        estimated_delay = int(delay_score * 10)  # Simple estimation
        
        return {
            "projectId": request.projectId,
            "delayProbability": round(delay_score, 2),
            "estimatedDelay": estimated_delay,
            "factors": factors,
            "recommendations": [
                "Increase resources" if resource_util > 0.9 else None,
                "Prioritize critical tasks" if high_priority_pending > 3 else None,
                "Review timeline" if delay_score > 0.6 else None
            ],
            "timestamp": datetime.utcnow().isoformat()
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/ml/alerts")
async def get_alerts():
    """
    Get recent AI-generated alerts
    """
    # Mock alerts - in production, fetch from database
    return [
        {
            "_id": "alert1",
            "type": "Risk Detection",
            "severity": "high",
            "message": "User john_doe shows high risk score (0.85)",
            "timestamp": datetime.utcnow().isoformat()
        },
        {
            "_id": "alert2",
            "type": "Burnout Detection",
            "severity": "medium",
            "message": "Employee jane_smith showing burnout signs",
            "timestamp": datetime.utcnow().isoformat()
        }
    ]

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "ML Analytics"}
```

## Common Patterns

### Protected API Calls (Frontend)

```javascript
// src/utils/api.js
import axios from 'axios';

const API_BASE = process.env.REACT_APP_API_URL;
const ML_API_BASE = process.env.REACT_APP_ML_API_URL;

export const apiClient = axios.create({
  baseURL: API_BASE,
  headers: {
    'Content-Type': 'application/json'
  }
});

export const mlApiClient = axios.create({
  baseURL: ML_API_BASE,
  headers: {
    'Content-Type': 'application/json'
  }
});

// Add token to requests
apiClient.interceptors.request.use(
  (config) => {
    const token = localStorage.getItem('token');
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
  },
  (error) => Promise.reject(error)
);

// Handle token expiration
apiClient.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 401) {
      localStorage.removeItem('token');
      window.location.href = '/login';
    }
    return Promise.reject(error);
  }
);

export default apiClient;
```

### Database Models

```javascript
// backend/models/Task.js
const mongoose = require('mongoose');

const taskSchema
