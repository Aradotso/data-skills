---
name: enterprise-user-management-system-ai
description: Full-stack user management system with AI-powered analytics, task tracking, ticket classification, and risk detection for enterprise applications
triggers:
  - "set up enterprise user management system"
  - "integrate AI analytics for user management"
  - "build user management dashboard with task tracking"
  - "implement ticket classification with machine learning"
  - "create role-based user administration system"
  - "add AI-powered risk detection to user app"
  - "deploy user management system with FastAPI ML service"
  - "configure JWT authentication for enterprise app"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System is a full-stack application that combines traditional user administration with AI-powered analytics. The system manages users, tasks, and support tickets while providing intelligent insights through machine learning models for risk detection, anomaly detection, burnout analysis, and predictive project management.

**Architecture:**
- **Frontend**: React.js for admin and user dashboards
- **Backend**: Node.js/Express REST API with JWT authentication
- **ML Service**: FastAPI with scikit-learn and River for online learning
- **Database**: MongoDB for data persistence

## Installation

### Prerequisites

```bash
# Required software
node --version  # v14+ required
python --version  # 3.8+ required
mongod --version  # MongoDB 4.4+ required
```

### Clone and Setup

```bash
# Clone repository
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics

# Install all services
# Backend
cd backend
npm install

# Frontend
cd ../frontend
npm install

# ML Service
cd ../ml-service
pip install -r requirements.txt
```

### Environment Configuration

**Backend (.env)**
```bash
# backend/.env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
JWT_SECRET=your_jwt_secret_here
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
```

**ML Service (.env)**
```bash
# ml-service/.env
MODEL_PATH=./models
DB_CONNECTION=mongodb://localhost:27017/enterprise_user_mgmt
LOG_LEVEL=INFO
```

**Frontend (.env)**
```bash
# frontend/.env
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
```

## Running the System

### Start All Services

```bash
# Terminal 1 - Backend
cd backend
npm start
# Runs on http://localhost:5000

# Terminal 2 - ML Service
cd ml-service
uvicorn main:app --reload --host 0.0.0.0 --port 8000
# Runs on http://localhost:8000

# Terminal 3 - Frontend
cd frontend
npm start
# Runs on http://localhost:3000
```

### Development Mode

```bash
# Backend with nodemon
cd backend
npm run dev

# ML Service with auto-reload
cd ml-service
uvicorn main:app --reload --log-level debug

# Frontend with hot reload
cd frontend
npm start
```

## Backend API Reference

### Authentication

```javascript
// Register new user
POST /api/auth/register
Content-Type: application/json

{
  "username": "john.doe",
  "email": "john@example.com",
  "password": "SecurePass123",
  "role": "user"  // "user" or "admin"
}

// Login
POST /api/auth/login
{
  "email": "john@example.com",
  "password": "SecurePass123"
}

// Response
{
  "token": "eyJhbGciOiJIUzI1NiIs...",
  "user": {
    "id": "507f1f77bcf86cd799439011",
    "username": "john.doe",
    "role": "user"
  }
}
```

### User Management (Admin Only)

```javascript
// Get all users
GET /api/users
Authorization: Bearer <token>

// Create user
POST /api/users
Authorization: Bearer <token>
{
  "username": "jane.smith",
  "email": "jane@example.com",
  "password": "TempPass123",
  "role": "user",
  "department": "Engineering"
}

// Update user
PUT /api/users/:userId
Authorization: Bearer <token>
{
  "role": "admin",
  "isActive": true
}

// Delete user
DELETE /api/users/:userId
Authorization: Bearer <token>
```

### Task Management

```javascript
// Get user tasks
GET /api/tasks
Authorization: Bearer <token>

// Create task
POST /api/tasks
Authorization: Bearer <token>
{
  "title": "Implement authentication",
  "description": "Add JWT-based auth",
  "assignedTo": "507f1f77bcf86cd799439011",
  "priority": "high",
  "dueDate": "2026-05-01T00:00:00Z",
  "status": "todo"  // todo, inprogress, done
}

// Update task status
PATCH /api/tasks/:taskId/status
{
  "status": "inprogress"
}

// Track time on task
POST /api/tasks/:taskId/time
{
  "duration": 3600  // seconds
}
```

### Support Tickets

```javascript
// Create ticket
POST /api/tickets
Authorization: Bearer <token>
{
  "title": "Cannot access dashboard",
  "description": "Getting 403 error when trying to access admin panel",
  "priority": "high",
  "category": "technical"
}

// Get user tickets
GET /api/tickets
Authorization: Bearer <token>

// Update ticket (Admin)
PATCH /api/tickets/:ticketId
{
  "status": "resolved",
  "resolution": "Fixed permissions issue"
}
```

## ML Service API Reference

### AI-Powered Features

```python
# ml-service/main.py structure
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
import joblib
import numpy as np

app = FastAPI()

class UserBehavior(BaseModel):
    user_id: str
    login_frequency: float
    task_completion_rate: float
    avg_task_duration: float
    ticket_count: int
    late_submissions: int
```

### Risk Detection

```python
# Predict user risk level
POST http://localhost:8000/api/ml/risk-detection
Content-Type: application/json

{
  "user_id": "507f1f77bcf86cd799439011",
  "login_frequency": 15,
  "task_completion_rate": 0.65,
  "avg_task_duration": 4200,
  "ticket_count": 8,
  "late_submissions": 3
}

# Response
{
  "user_id": "507f1f77bcf86cd799439011",
  "risk_level": "medium",
  "risk_score": 0.68,
  "factors": [
    "Low task completion rate",
    "High ticket count"
  ],
  "recommendations": [
    "Review workload distribution",
    "Provide additional training"
  ]
}
```

### Ticket Classification

```python
# Classify support ticket
POST http://localhost:8000/api/ml/classify-ticket
{
  "title": "Cannot access dashboard",
  "description": "Getting 403 error when trying to access admin panel",
  "user_history": {
    "previous_tickets": 2,
    "avg_resolution_time": 3600
  }
}

# Response
{
  "category": "technical",
  "priority": "high",
  "estimated_resolution_time": 7200,
  "suggested_assignee": "tech_support_team",
  "confidence": 0.92
}
```

### Burnout Detection

```python
# Detect employee burnout risk
POST http://localhost:8000/api/ml/burnout-detection
{
  "user_id": "507f1f77bcf86cd799439011",
  "work_hours_per_week": 65,
  "completed_tasks": 28,
  "missed_deadlines": 5,
  "overtime_hours": 15,
  "vacation_days_unused": 12
}

# Response
{
  "burnout_risk": "high",
  "score": 0.82,
  "indicators": [
    "Excessive work hours",
    "High missed deadline rate",
    "No vacation taken"
  ],
  "suggestions": [
    "Schedule mandatory time off",
    "Redistribute workload",
    "Schedule wellness check-in"
  ]
}
```

### Anomaly Detection

```python
# Detect unusual user behavior
POST http://localhost:8000/api/ml/anomaly-detection
{
  "user_id": "507f1f77bcf86cd799439011",
  "login_time": "2026-04-15T03:45:00Z",
  "login_location": "Unknown",
  "actions": [
    {"type": "data_export", "count": 15},
    {"type": "permission_change", "count": 3}
  ]
}

# Response
{
  "is_anomaly": true,
  "anomaly_score": 0.89,
  "flags": [
    "Unusual login time",
    "High-risk action frequency",
    "Unrecognized location"
  ],
  "recommended_action": "Require additional authentication"
}
```

## Frontend Integration

### React Component Example - User Dashboard

```javascript
// frontend/src/components/UserDashboard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const UserDashboard = () => {
  const [tasks, setTasks] = useState([]);
  const [aiInsights, setAiInsights] = useState(null);
  const token = localStorage.getItem('token');

  useEffect(() => {
    fetchTasks();
    fetchAIInsights();
  }, []);

  const fetchTasks = async () => {
    try {
      const response = await axios.get(
        `${process.env.REACT_APP_API_URL}/api/tasks`,
        {
          headers: { Authorization: `Bearer ${token}` }
        }
      );
      setTasks(response.data);
    } catch (error) {
      console.error('Error fetching tasks:', error);
    }
  };

  const fetchAIInsights = async () => {
    try {
      const response = await axios.get(
        `${process.env.REACT_APP_API_URL}/api/users/insights`,
        {
          headers: { Authorization: `Bearer ${token}` }
        }
      );
      setAiInsights(response.data);
    } catch (error) {
      console.error('Error fetching AI insights:', error);
    }
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    try {
      await axios.patch(
        `${process.env.REACT_APP_API_URL}/api/tasks/${taskId}/status`,
        { status: newStatus },
        {
          headers: { Authorization: `Bearer ${token}` }
        }
      );
      fetchTasks();
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  return (
    <div className="dashboard">
      <h1>My Dashboard</h1>
      
      {aiInsights && aiInsights.risk_level && (
        <div className={`alert alert-${aiInsights.risk_level}`}>
          <h3>AI Insights</h3>
          <p>Risk Level: {aiInsights.risk_level}</p>
          <ul>
            {aiInsights.recommendations.map((rec, idx) => (
              <li key={idx}>{rec}</li>
            ))}
          </ul>
        </div>
      )}

      <div className="kanban-board">
        <div className="column">
          <h2>To Do</h2>
          {tasks.filter(t => t.status === 'todo').map(task => (
            <TaskCard 
              key={task._id} 
              task={task} 
              onStatusChange={updateTaskStatus}
            />
          ))}
        </div>
        
        <div className="column">
          <h2>In Progress</h2>
          {tasks.filter(t => t.status === 'inprogress').map(task => (
            <TaskCard 
              key={task._id} 
              task={task} 
              onStatusChange={updateTaskStatus}
            />
          ))}
        </div>
        
        <div className="column">
          <h2>Done</h2>
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

const TaskCard = ({ task, onStatusChange }) => (
  <div className="task-card">
    <h3>{task.title}</h3>
    <p>{task.description}</p>
    <select 
      value={task.status} 
      onChange={(e) => onStatusChange(task._id, e.target.value)}
    >
      <option value="todo">To Do</option>
      <option value="inprogress">In Progress</option>
      <option value="done">Done</option>
    </select>
  </div>
);

export default UserDashboard;
```

### Admin Analytics Component

```javascript
// frontend/src/components/AdminAnalytics.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import { LineChart, Line, XAxis, YAxis, CartesianGrid, Tooltip } from 'recharts';

const AdminAnalytics = () => {
  const [analytics, setAnalytics] = useState(null);
  const [riskUsers, setRiskUsers] = useState([]);
  const token = localStorage.getItem('token');

  useEffect(() => {
    fetchAnalytics();
    fetchRiskUsers();
  }, []);

  const fetchAnalytics = async () => {
    try {
      const response = await axios.get(
        `${process.env.REACT_APP_API_URL}/api/admin/analytics`,
        {
          headers: { Authorization: `Bearer ${token}` }
        }
      );
      setAnalytics(response.data);
    } catch (error) {
      console.error('Error fetching analytics:', error);
    }
  };

  const fetchRiskUsers = async () => {
    try {
      const response = await axios.get(
        `${process.env.REACT_APP_ML_API_URL}/api/ml/high-risk-users`,
        {
          headers: { Authorization: `Bearer ${token}` }
        }
      );
      setRiskUsers(response.data);
    } catch (error) {
      console.error('Error fetching risk users:', error);
    }
  };

  return (
    <div className="admin-analytics">
      <h1>Organization Analytics</h1>
      
      {analytics && (
        <div className="metrics">
          <div className="metric-card">
            <h3>Total Users</h3>
            <p className="metric-value">{analytics.totalUsers}</p>
          </div>
          <div className="metric-card">
            <h3>Active Tasks</h3>
            <p className="metric-value">{analytics.activeTasks}</p>
          </div>
          <div className="metric-card">
            <h3>Open Tickets</h3>
            <p className="metric-value">{analytics.openTickets}</p>
          </div>
          <div className="metric-card">
            <h3>Completion Rate</h3>
            <p className="metric-value">{analytics.completionRate}%</p>
          </div>
        </div>
      )}

      <div className="risk-alerts">
        <h2>High-Risk Users</h2>
        <table>
          <thead>
            <tr>
              <th>User</th>
              <th>Risk Level</th>
              <th>Score</th>
              <th>Primary Factor</th>
              <th>Action</th>
            </tr>
          </thead>
          <tbody>
            {riskUsers.map(user => (
              <tr key={user.user_id}>
                <td>{user.username}</td>
                <td>
                  <span className={`badge badge-${user.risk_level}`}>
                    {user.risk_level}
                  </span>
                </td>
                <td>{(user.risk_score * 100).toFixed(1)}%</td>
                <td>{user.factors[0]}</td>
                <td>
                  <button onClick={() => handleIntervention(user.user_id)}>
                    Take Action
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

## Backend Implementation Patterns

### User Authentication Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

const authenticate = async (req, res, next) => {
  try {
    const token = req.header('Authorization')?.replace('Bearer ', '');
    
    if (!token) {
      return res.status(401).json({ error: 'Authentication required' });
    }

    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    const user = await User.findById(decoded.userId);

    if (!user || !user.isActive) {
      return res.status(401).json({ error: 'Invalid authentication' });
    }

    req.user = user;
    req.token = token;
    next();
  } catch (error) {
    res.status(401).json({ error: 'Please authenticate' });
  }
};

const requireAdmin = (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ error: 'Admin access required' });
  }
  next();
};

module.exports = { authenticate, requireAdmin };
```

### Task Controller with AI Integration

```javascript
// backend/controllers/taskController.js
const Task = require('../models/Task');
const axios = require('axios');

const createTask = async (req, res) => {
  try {
    const { title, description, assignedTo, priority, dueDate } = req.body;
    
    const task = new Task({
      title,
      description,
      assignedTo,
      priority,
      dueDate,
      createdBy: req.user._id,
      status: 'todo'
    });

    await task.save();

    // Get AI prediction for task complexity
    try {
      const mlResponse = await axios.post(
        `${process.env.ML_SERVICE_URL}/api/ml/predict-task-duration`,
        {
          title,
          description,
          priority,
          assignedTo
        }
      );
      
      task.estimatedDuration = mlResponse.data.estimated_duration;
      await task.save();
    } catch (mlError) {
      console.log('ML service unavailable, continuing without prediction');
    }

    res.status(201).json(task);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
};

const getUserTasks = async (req, res) => {
  try {
    const tasks = await Task.find({ 
      assignedTo: req.user._id 
    })
    .populate('createdBy', 'username email')
    .sort({ createdAt: -1 });

    res.json(tasks);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
};

const updateTaskStatus = async (req, res) => {
  try {
    const { taskId } = req.params;
    const { status } = req.body;

    const task = await Task.findOne({ 
      _id: taskId, 
      assignedTo: req.user._id 
    });

    if (!task) {
      return res.status(404).json({ error: 'Task not found' });
    }

    task.status = status;
    if (status === 'done') {
      task.completedAt = new Date();
    }

    await task.save();
    res.json(task);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
};

module.exports = { createTask, getUserTasks, updateTaskStatus };
```

### Ticket Management with AI Classification

```javascript
// backend/controllers/ticketController.js
const Ticket = require('../models/Ticket');
const axios = require('axios');

const createTicket = async (req, res) => {
  try {
    const { title, description, priority, category } = req.body;

    // Get AI classification if category not provided
    let ticketCategory = category;
    let aiPriority = priority;

    if (!category || !priority) {
      try {
        const mlResponse = await axios.post(
          `${process.env.ML_SERVICE_URL}/api/ml/classify-ticket`,
          {
            title,
            description,
            user_history: {
              previous_tickets: await Ticket.countDocuments({ 
                createdBy: req.user._id 
              })
            }
          }
        );

        ticketCategory = mlResponse.data.category;
        aiPriority = mlResponse.data.priority;
      } catch (mlError) {
        console.log('ML classification unavailable');
        ticketCategory = category || 'general';
        aiPriority = priority || 'medium';
      }
    }

    const ticket = new Ticket({
      title,
      description,
      priority: aiPriority,
      category: ticketCategory,
      createdBy: req.user._id,
      status: 'open'
    });

    await ticket.save();
    res.status(201).json(ticket);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
};

module.exports = { createTicket };
```

## ML Service Implementation

### Risk Detection Model

```python
# ml-service/models/risk_detector.py
import numpy as np
from sklearn.ensemble import RandomForestClassifier
import joblib
import os

class RiskDetector:
    def __init__(self, model_path='./models/risk_model.pkl'):
        self.model_path = model_path
        if os.path.exists(model_path):
            self.model = joblib.load(model_path)
        else:
            self.model = RandomForestClassifier(n_estimators=100, random_state=42)
    
    def predict_risk(self, features):
        """
        Features expected:
        - login_frequency: int (logins per week)
        - task_completion_rate: float (0-1)
        - avg_task_duration: float (seconds)
        - ticket_count: int
        - late_submissions: int
        """
        feature_array = np.array([[
            features['login_frequency'],
            features['task_completion_rate'],
            features['avg_task_duration'],
            features['ticket_count'],
            features['late_submissions']
        ]])
        
        risk_score = self.model.predict_proba(feature_array)[0][1]
        
        if risk_score > 0.7:
            risk_level = 'high'
        elif risk_score > 0.4:
            risk_level = 'medium'
        else:
            risk_level = 'low'
        
        factors = self._identify_risk_factors(features)
        recommendations = self._generate_recommendations(factors, risk_level)
        
        return {
            'risk_level': risk_level,
            'risk_score': float(risk_score),
            'factors': factors,
            'recommendations': recommendations
        }
    
    def _identify_risk_factors(self, features):
        factors = []
        
        if features['task_completion_rate'] < 0.7:
            factors.append('Low task completion rate')
        
        if features['late_submissions'] > 3:
            factors.append('High number of late submissions')
        
        if features['ticket_count'] > 5:
            factors.append('High ticket count')
        
        if features['login_frequency'] < 5:
            factors.append('Low engagement')
        
        return factors
    
    def _generate_recommendations(self, factors, risk_level):
        recommendations = []
        
        if risk_level == 'high':
            recommendations.append('Schedule immediate one-on-one meeting')
            recommendations.append('Review workload distribution')
        
        if 'Low task completion rate' in factors:
            recommendations.append('Provide additional training or support')
        
        if 'High number of late submissions' in factors:
            recommendations.append('Assess task complexity and deadlines')
        
        if 'High ticket count' in factors:
            recommendations.append('Address recurring technical issues')
        
        return recommendations
    
    def save_model(self):
        joblib.dump(self.model, self.model_path)
```

### FastAPI Endpoints

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from typing import List, Optional
from models.risk_detector import RiskDetector
from models.ticket_classifier import TicketClassifier
from models.burnout_detector import BurnoutDetector

app = FastAPI(title="Enterprise ML Service", version="1.0.0")

# CORS configuration
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Initialize models
risk_detector = RiskDetector()
ticket_classifier = TicketClassifier()
burnout_detector = BurnoutDetector()

class UserBehavior(BaseModel):
    user_id: str
    login_frequency: int
    task_completion_rate: float
    avg_task_duration: float
    ticket_count: int
    late_submissions: int

class TicketData(BaseModel):
    title: str
    description: str
    user_history: dict

class BurnoutData(BaseModel):
    user_id: str
    work_hours_per_week: float
    completed_tasks: int
    missed_deadlines: int
    overtime_hours: float
    vacation_days_unused: int

@app.get("/")
def read_root():
    return {"message": "Enterprise ML Service API", "version": "1.0.0"}

@app.post("/api/ml/risk-detection")
def detect_risk(data: UserBehavior):
    """Predict user risk level based on behavior patterns"""
    try:
        result = risk_detector.predict_risk({
            'login_frequency': data.login_frequency,
            'task_completion_rate': data.task_completion_rate,
            'avg_task_duration': data.avg_task_duration,
            'ticket_count': data.ticket_count,
            'late_submissions': data.late_submissions
        })
        
        result['user_id'] = data.user_id
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/classify-ticket")
def classify_ticket(data: TicketData):
    """Classify support ticket and predict priority"""
    try:
        result = ticket_classifier.classify({
            'title': data.title,
            'description': data.description,
            'user_history': data.user_history
        })
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/burnout-detection")
def detect_burnout(data: BurnoutData):
    """Detect employee burnout risk"""
    try:
        result = burnout_detector.predict({
            'work_hours_per_week': data.work_hours_per_week,
            'completed_tasks': data.completed_tasks,
            'missed_deadlines': data.missed_deadlines,
            'overtime_hours': data.overtime_hours,
            'vacation_days_unused': data.vacation_days_unused
        })
        
        result['user_id'] = data.user_id
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/anomaly-detection")
def detect_anomaly(data: dict):
    """Detect unusual user behavior patterns"""
    try:
        # Implement anomaly detection logic
        from datetime import datetime
        
        login_time = datetime.fromisoformat(data['login_time'].replace('Z', '+00:00'))
        hour = login_time.hour
        
        is_unusual_time = hour < 6 or hour > 22
        high_risk_actions = sum(action['count'] for action in data['actions'])
        
        anomaly_score = 0.0
        flags = []
        
        if is_unusual_time:
            anomaly_score += 0.3
            flags.append('Unusual login time')
        
        if high_risk_actions > 10:
            anomaly_score += 0.4
            flags.append('High-risk action frequency')
        
        if data.get('login_location') == 'Unknown':
            anomaly_score += 0.3
            flags.append('Unrecognized location')
        
        is_anomaly = anomaly_score > 0.5
        
        return {
            'is_anomaly': is_anomaly,
            'anomaly_score': min(anomaly_score, 1.0),
            'flags': flags,
            'recommended_action': 'Require additional authentication' if is_anomaly else 'No action needed'
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

## Database Models

### User Model (MongoDB/Mongoose)

```javascript
// backend/models/User.js
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');

const userSchema = new mongoose.Schema({
  username: {
    type: String,
    required: true,
    unique: true,
    trim: true
  },
  email: {
    type: String,
    required: true,
    unique: true,
    lowercase: true,
    trim: true
  },
  password: {
    type: String,
    required: true,
    minlength: 6
  },
  role: {
    type: String,
    enum: ['user', 'admin'],
    default: 'user'
  },
  department: {
    type: String,
    default: 'General'
  },
  isActive: {
    type: Boolean,
    default: true
  },
  
