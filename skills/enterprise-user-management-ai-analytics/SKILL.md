---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics, task tracking, and intelligent ticket routing using React, Node.js, and FastAPI ML service
triggers:
  - "set up enterprise user management system"
  - "integrate AI analytics for user management"
  - "configure user dashboard with task tracking"
  - "implement AI ticket classification"
  - "build admin panel with user management"
  - "add burnout detection to user system"
  - "create kanban board for task management"
  - "deploy user management with ML service"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This is a full-stack enterprise user management system that combines traditional CRUD operations with AI-powered analytics including risk detection, anomaly detection, burnout analysis, and intelligent ticket routing. The system features role-based access control (admin/user), task management with Kanban boards, support ticket systems, and ML-driven insights.

## Architecture Overview

The system consists of three main components:

1. **Frontend** (React.js) - User/Admin dashboards, task boards, ticket management
2. **Backend** (Node.js/Express) - REST API, authentication, business logic
3. **ML Service** (FastAPI) - AI models for predictions, classifications, anomaly detection

## Installation

### Prerequisites

```bash
# Required
node >= 14.x
npm >= 6.x
python >= 3.8
mongodb >= 4.x
```

### Full System Setup

```bash
# Clone repository
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics

# Backend setup
cd backend
npm install
cp .env.example .env  # Configure MongoDB URI, JWT secret

# ML Service setup
cd ../ml-service
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate
pip install -r requirements.txt

# Frontend setup
cd ../frontend
npm install
```

### Environment Configuration

**Backend** (`backend/.env`):
```env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
JWT_SECRET=your_jwt_secret_here
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
```

**ML Service** (`ml-service/.env`):
```env
MODEL_PATH=./models
LOG_LEVEL=INFO
BACKEND_URL=http://localhost:5000
```

**Frontend** (`frontend/.env`):
```env
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_API_URL=http://localhost:8000
```

## Running the System

### Start All Services

```bash
# Terminal 1 - Backend
cd backend
npm start
# Runs at http://localhost:5000

# Terminal 2 - ML Service
cd ml-service
uvicorn main:app --reload --port 8000
# Runs at http://localhost:8000

# Terminal 3 - Frontend
cd frontend
npm start
# Runs at http://localhost:3000
```

### Production Build

```bash
# Frontend production build
cd frontend
npm run build

# Serve with backend
cd ../backend
npm run build
npm run start:prod
```

## Backend API Reference

### Authentication

```javascript
// Login
POST /api/auth/login
Content-Type: application/json

{
  "email": "user@example.com",
  "password": "password123"
}

// Response
{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "user": {
    "id": "user_id",
    "email": "user@example.com",
    "role": "user"
  }
}

// Register
POST /api/auth/register
Content-Type: application/json

{
  "name": "John Doe",
  "email": "john@example.com",
  "password": "password123",
  "role": "user"
}
```

### User Management (Admin)

```javascript
// Get all users
GET /api/users
Authorization: Bearer <token>

// Create user
POST /api/users
Authorization: Bearer <token>
Content-Type: application/json

{
  "name": "Jane Smith",
  "email": "jane@example.com",
  "role": "user",
  "department": "Engineering"
}

// Update user
PUT /api/users/:userId
Authorization: Bearer <token>
Content-Type: application/json

{
  "name": "Jane Smith Updated",
  "status": "active"
}

// Delete user
DELETE /api/users/:userId
Authorization: Bearer <token>
```

### Task Management

```javascript
// Get user tasks
GET /api/tasks/user/:userId
Authorization: Bearer <token>

// Create task
POST /api/tasks
Authorization: Bearer <token>
Content-Type: application/json

{
  "title": "Complete API documentation",
  "description": "Document all REST endpoints",
  "assignedTo": "user_id",
  "priority": "high",
  "status": "todo",
  "dueDate": "2026-05-01"
}

// Update task status
PATCH /api/tasks/:taskId/status
Authorization: Bearer <token>
Content-Type: application/json

{
  "status": "in-progress"
}

// Track time
POST /api/tasks/:taskId/time
Authorization: Bearer <token>
Content-Type: application/json

{
  "timeSpent": 3600  // seconds
}
```

### Support Tickets

```javascript
// Create ticket
POST /api/tickets
Authorization: Bearer <token>
Content-Type: application/json

{
  "title": "Login issue",
  "description": "Cannot login with correct credentials",
  "priority": "high",
  "category": "technical"
}

// Get tickets (admin - all, user - own)
GET /api/tickets
Authorization: Bearer <token>

// Update ticket
PUT /api/tickets/:ticketId
Authorization: Bearer <token>
Content-Type: application/json

{
  "status": "resolved",
  "assignedTo": "admin_id"
}
```

## ML Service API Reference

### AI Ticket Classification

```javascript
// Classify ticket
POST /ml/classify-ticket
Content-Type: application/json

{
  "title": "Cannot access dashboard",
  "description": "Getting 404 error when trying to access /dashboard",
  "priority": "high"
}

// Response
{
  "category": "technical",
  "suggested_priority": "high",
  "confidence": 0.89,
  "suggested_assignment": "tech_support_team"
}
```

### Risk Prediction

```javascript
// Predict user risk
POST /ml/predict-risk
Content-Type: application/json

{
  "userId": "user_123",
  "loginAttempts": 5,
  "failedLogins": 3,
  "activityHours": [22, 23, 1, 2],
  "dataAccessed": 150,
  "unusualLocations": 2
}

// Response
{
  "riskScore": 0.78,
  "riskLevel": "high",
  "factors": [
    "Multiple failed login attempts",
    "Activity during unusual hours",
    "Access from unusual locations"
  ],
  "recommendation": "Enable MFA and review recent activity"
}
```

### Anomaly Detection

```javascript
// Detect anomalies
POST /ml/detect-anomaly
Content-Type: application/json

{
  "userId": "user_123",
  "features": {
    "loginTime": "03:00:00",
    "location": "Unknown",
    "deviceFingerprint": "new_device_xyz",
    "dataVolume": 500
  }
}

// Response
{
  "isAnomaly": true,
  "anomalyScore": 0.92,
  "anomalyType": "unusual_access_pattern",
  "details": "Login from new device at unusual time"
}
```

### Burnout Detection

```javascript
// Analyze burnout risk
POST /ml/burnout-analysis
Content-Type: application/json

{
  "userId": "user_123",
  "tasksCompleted": 45,
  "avgWorkHours": 11.5,
  "weekendWork": 3,
  "missedDeadlines": 5,
  "ticketsRaised": 8
}

// Response
{
  "burnoutScore": 0.85,
  "burnoutLevel": "high",
  "indicators": [
    "Excessive work hours (11.5 avg)",
    "Frequent weekend work",
    "Multiple missed deadlines"
  ],
  "recommendations": [
    "Reduce workload",
    "Consider time off",
    "Redistribute tasks"
  ]
}
```

### Predictive Project Insights

```javascript
// Predict project delay
POST /ml/predict-delay
Content-Type: application/json

{
  "projectId": "proj_123",
  "totalTasks": 50,
  "completedTasks": 15,
  "daysElapsed": 30,
  "totalDays": 90,
  "teamSize": 5,
  "avgTaskCompletionTime": 2.5
}

// Response
{
  "delayProbability": 0.72,
  "expectedDelay": 15,
  "delayReason": "Current completion rate below target",
  "suggestedActions": [
    "Add 2 more team members",
    "Reduce scope by 10 tasks",
    "Extend deadline by 2 weeks"
  ]
}
```

## Frontend Integration Examples

### React Authentication Hook

```javascript
// hooks/useAuth.js
import { useState, useEffect, createContext, useContext } from 'react';
import axios from 'axios';

const AuthContext = createContext();

export const AuthProvider = ({ children }) => {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);

  const API_URL = process.env.REACT_APP_API_URL;

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
      const response = await axios.get(`${API_URL}/auth/me`);
      setUser(response.data);
    } catch (error) {
      localStorage.removeItem('token');
    } finally {
      setLoading(false);
    }
  };

  const login = async (email, password) => {
    const response = await axios.post(`${API_URL}/auth/login`, {
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

  return (
    <AuthContext.Provider value={{ user, login, logout, loading }}>
      {children}
    </AuthContext.Provider>
  );
};

export const useAuth = () => useContext(AuthContext);
```

### Task Management Component

```javascript
// components/TaskBoard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const TaskBoard = ({ userId }) => {
  const [tasks, setTasks] = useState({
    todo: [],
    'in-progress': [],
    done: []
  });

  const API_URL = process.env.REACT_APP_API_URL;

  useEffect(() => {
    fetchTasks();
  }, [userId]);

  const fetchTasks = async () => {
    try {
      const response = await axios.get(`${API_URL}/tasks/user/${userId}`);
      
      const categorized = {
        todo: [],
        'in-progress': [],
        done: []
      };
      
      response.data.forEach(task => {
        categorized[task.status].push(task);
      });
      
      setTasks(categorized);
    } catch (error) {
      console.error('Error fetching tasks:', error);
    }
  };

  const moveTask = async (taskId, newStatus) => {
    try {
      await axios.patch(`${API_URL}/tasks/${taskId}/status`, {
        status: newStatus
      });
      
      fetchTasks(); // Refresh
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  const onDragStart = (e, taskId, currentStatus) => {
    e.dataTransfer.setData('taskId', taskId);
    e.dataTransfer.setData('currentStatus', currentStatus);
  };

  const onDrop = (e, newStatus) => {
    const taskId = e.dataTransfer.getData('taskId');
    const currentStatus = e.dataTransfer.getData('currentStatus');
    
    if (currentStatus !== newStatus) {
      moveTask(taskId, newStatus);
    }
  };

  const onDragOver = (e) => {
    e.preventDefault();
  };

  return (
    <div className="task-board">
      {['todo', 'in-progress', 'done'].map(status => (
        <div
          key={status}
          className="task-column"
          onDrop={(e) => onDrop(e, status)}
          onDragOver={onDragOver}
        >
          <h3>{status.toUpperCase().replace('-', ' ')}</h3>
          {tasks[status].map(task => (
            <div
              key={task._id}
              className="task-card"
              draggable
              onDragStart={(e) => onDragStart(e, task._id, status)}
            >
              <h4>{task.title}</h4>
              <p>{task.description}</p>
              <span className={`priority ${task.priority}`}>
                {task.priority}
              </span>
            </div>
          ))}
        </div>
      ))}
    </div>
  );
};

export default TaskBoard;
```

### AI Analytics Dashboard

```javascript
// components/AIAnalytics.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const AIAnalytics = ({ userId }) => {
  const [analytics, setAnalytics] = useState({
    risk: null,
    burnout: null,
    anomalies: []
  });
  const [loading, setLoading] = useState(true);

  const ML_API_URL = process.env.REACT_APP_ML_API_URL;
  const API_URL = process.env.REACT_APP_API_URL;

  useEffect(() => {
    fetchAnalytics();
  }, [userId]);

  const fetchAnalytics = async () => {
    try {
      setLoading(true);
      
      // Fetch user data
      const userDataResponse = await axios.get(`${API_URL}/users/${userId}/analytics`);
      const userData = userDataResponse.data;
      
      // Get risk prediction
      const riskResponse = await axios.post(`${ML_API_URL}/ml/predict-risk`, {
        userId: userId,
        loginAttempts: userData.loginAttempts,
        failedLogins: userData.failedLogins,
        activityHours: userData.activityHours,
        dataAccessed: userData.dataAccessed,
        unusualLocations: userData.unusualLocations
      });
      
      // Get burnout analysis
      const burnoutResponse = await axios.post(`${ML_API_URL}/ml/burnout-analysis`, {
        userId: userId,
        tasksCompleted: userData.tasksCompleted,
        avgWorkHours: userData.avgWorkHours,
        weekendWork: userData.weekendWork,
        missedDeadlines: userData.missedDeadlines,
        ticketsRaised: userData.ticketsRaised
      });
      
      setAnalytics({
        risk: riskResponse.data,
        burnout: burnoutResponse.data,
        anomalies: userData.recentAnomalies || []
      });
      
    } catch (error) {
      console.error('Error fetching analytics:', error);
    } finally {
      setLoading(false);
    }
  };

  if (loading) return <div>Loading analytics...</div>;

  return (
    <div className="ai-analytics">
      <div className="analytics-card">
        <h3>Risk Assessment</h3>
        <div className={`risk-level ${analytics.risk?.riskLevel}`}>
          {analytics.risk?.riskLevel?.toUpperCase()}
        </div>
        <p>Score: {(analytics.risk?.riskScore * 100).toFixed(0)}%</p>
        <ul>
          {analytics.risk?.factors.map((factor, idx) => (
            <li key={idx}>{factor}</li>
          ))}
        </ul>
      </div>

      <div className="analytics-card">
        <h3>Burnout Detection</h3>
        <div className={`burnout-level ${analytics.burnout?.burnoutLevel}`}>
          {analytics.burnout?.burnoutLevel?.toUpperCase()}
        </div>
        <p>Score: {(analytics.burnout?.burnoutScore * 100).toFixed(0)}%</p>
        <h4>Recommendations:</h4>
        <ul>
          {analytics.burnout?.recommendations.map((rec, idx) => (
            <li key={idx}>{rec}</li>
          ))}
        </ul>
      </div>

      <div className="analytics-card">
        <h3>Recent Anomalies</h3>
        {analytics.anomalies.length === 0 ? (
          <p>No anomalies detected</p>
        ) : (
          <ul>
            {analytics.anomalies.map((anomaly, idx) => (
              <li key={idx}>
                <strong>{anomaly.type}:</strong> {anomaly.details}
              </li>
            ))}
          </ul>
        )}
      </div>
    </div>
  );
};

export default AIAnalytics;
```

### Smart Ticket Creation

```javascript
// components/CreateTicket.jsx
import React, { useState } from 'react';
import axios from 'axios';

const CreateTicket = ({ userId, onTicketCreated }) => {
  const [formData, setFormData] = useState({
    title: '',
    description: '',
    priority: 'medium'
  });
  const [aiSuggestions, setAiSuggestions] = useState(null);
  const [loading, setLoading] = useState(false);

  const API_URL = process.env.REACT_APP_API_URL;
  const ML_API_URL = process.env.REACT_APP_ML_API_URL;

  const handleChange = (e) => {
    setFormData({
      ...formData,
      [e.target.name]: e.target.value
    });
  };

  const getAISuggestions = async () => {
    if (!formData.title || !formData.description) return;
    
    try {
      const response = await axios.post(`${ML_API_URL}/ml/classify-ticket`, {
        title: formData.title,
        description: formData.description,
        priority: formData.priority
      });
      
      setAiSuggestions(response.data);
    } catch (error) {
      console.error('Error getting AI suggestions:', error);
    }
  };

  const handleSubmit = async (e) => {
    e.preventDefault();
    setLoading(true);

    try {
      const ticketData = {
        ...formData,
        userId: userId,
        category: aiSuggestions?.category || 'general',
        suggestedPriority: aiSuggestions?.suggested_priority
      };

      const response = await axios.post(`${API_URL}/tickets`, ticketData);
      
      onTicketCreated(response.data);
      
      // Reset form
      setFormData({
        title: '',
        description: '',
        priority: 'medium'
      });
      setAiSuggestions(null);
      
    } catch (error) {
      console.error('Error creating ticket:', error);
    } finally {
      setLoading(false);
    }
  };

  return (
    <div className="create-ticket">
      <h2>Create Support Ticket</h2>
      
      <form onSubmit={handleSubmit}>
        <div className="form-group">
          <label>Title</label>
          <input
            type="text"
            name="title"
            value={formData.title}
            onChange={handleChange}
            onBlur={getAISuggestions}
            required
          />
        </div>

        <div className="form-group">
          <label>Description</label>
          <textarea
            name="description"
            value={formData.description}
            onChange={handleChange}
            onBlur={getAISuggestions}
            rows="5"
            required
          />
        </div>

        <div className="form-group">
          <label>Priority</label>
          <select
            name="priority"
            value={formData.priority}
            onChange={handleChange}
          >
            <option value="low">Low</option>
            <option value="medium">Medium</option>
            <option value="high">High</option>
          </select>
        </div>

        {aiSuggestions && (
          <div className="ai-suggestions">
            <h3>AI Suggestions</h3>
            <p><strong>Category:</strong> {aiSuggestions.category}</p>
            <p><strong>Suggested Priority:</strong> {aiSuggestions.suggested_priority}</p>
            <p><strong>Confidence:</strong> {(aiSuggestions.confidence * 100).toFixed(0)}%</p>
            {aiSuggestions.suggested_assignment && (
              <p><strong>Suggested Team:</strong> {aiSuggestions.suggested_assignment}</p>
            )}
          </div>
        )}

        <button type="submit" disabled={loading}>
          {loading ? 'Creating...' : 'Create Ticket'}
        </button>
      </form>
    </div>
  );
};

export default CreateTicket;
```

## Backend Implementation Patterns

### JWT Authentication Middleware

```javascript
// middleware/auth.js
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

    if (!user) {
      return res.status(401).json({ error: 'User not found' });
    }

    req.user = user;
    req.token = token;
    next();
    
  } catch (error) {
    res.status(401).json({ error: 'Invalid token' });
  }
};

const authorize = (...roles) => {
  return (req, res, next) => {
    if (!roles.includes(req.user.role)) {
      return res.status(403).json({ 
        error: 'Access denied. Insufficient permissions.' 
      });
    }
    next();
  };
};

module.exports = { authenticate, authorize };
```

### User Controller

```javascript
// controllers/userController.js
const User = require('../models/User');
const bcrypt = require('bcryptjs');

exports.getAllUsers = async (req, res) => {
  try {
    const users = await User.find()
      .select('-password')
      .sort({ createdAt: -1 });
    
    res.json(users);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
};

exports.createUser = async (req, res) => {
  try {
    const { name, email, password, role, department } = req.body;

    // Check if user exists
    const existingUser = await User.findOne({ email });
    if (existingUser) {
      return res.status(400).json({ error: 'User already exists' });
    }

    // Hash password
    const hashedPassword = await bcrypt.hash(password, 10);

    const user = new User({
      name,
      email,
      password: hashedPassword,
      role: role || 'user',
      department
    });

    await user.save();

    res.status(201).json({
      message: 'User created successfully',
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

exports.updateUser = async (req, res) => {
  try {
    const { userId } = req.params;
    const updates = req.body;

    // Don't allow password updates through this endpoint
    delete updates.password;

    const user = await User.findByIdAndUpdate(
      userId,
      { $set: updates },
      { new: true, runValidators: true }
    ).select('-password');

    if (!user) {
      return res.status(404).json({ error: 'User not found' });
    }

    res.json(user);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
};

exports.deleteUser = async (req, res) => {
  try {
    const { userId } = req.params;

    const user = await User.findByIdAndDelete(userId);

    if (!user) {
      return res.status(404).json({ error: 'User not found' });
    }

    res.json({ message: 'User deleted successfully' });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
};

exports.getUserAnalytics = async (req, res) => {
  try {
    const { userId } = req.params;
    const user = await User.findById(userId);

    if (!user) {
      return res.status(404).json({ error: 'User not found' });
    }

    // Aggregate data from various sources
    const analytics = {
      loginAttempts: user.loginHistory?.length || 0,
      failedLogins: user.loginHistory?.filter(l => !l.success).length || 0,
      activityHours: user.activityLog?.map(a => new Date(a.timestamp).getHours()) || [],
      dataAccessed: user.dataAccessLog?.length || 0,
      unusualLocations: user.loginHistory?.filter(l => l.unusual).length || 0,
      tasksCompleted: await Task.countDocuments({ 
        assignedTo: userId, 
        status: 'done' 
      }),
      avgWorkHours: user.workHoursLog?.reduce((a, b) => a + b, 0) / 7 || 0,
      weekendWork: user.workHoursLog?.weekends || 0,
      missedDeadlines: await Task.countDocuments({
        assignedTo: userId,
        dueDate: { $lt: new Date() },
        status: { $ne: 'done' }
      }),
      ticketsRaised: await Ticket.countDocuments({ createdBy: userId })
    };

    res.json(analytics);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
};
```

### Task Controller

```javascript
// controllers/taskController.js
const Task = require('../models/Task');
const axios = require('axios');

exports.createTask = async (req, res) => {
  try {
    const { title, description, assignedTo, priority, dueDate } = req.body;

    const task = new Task({
      title,
      description,
      assignedTo,
      priority: priority || 'medium',
      status: 'todo',
      dueDate,
      createdBy: req.user._id
    });

    await task.save();

    res.status(201).json(task);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
};

exports.getUserTasks = async (req, res) => {
  try {
    const { userId } = req.params;

    const tasks = await Task.find({ assignedTo: userId })
      .populate('createdBy', 'name email')
      .sort({ createdAt: -1 });

    res.json(tasks);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
};

exports.updateTaskStatus = async (req, res) => {
  try {
    const { taskId } = req.params;
    const { status } = req.body;

    const task = await Task.findByIdAndUpdate(
      taskId,
      { 
        status,
        ...(status === 'done' && { completedAt: new Date() })
      },
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
    const { taskId } = req.params;
    const { timeSpent } = req.body;

    const task = await Task.findById(taskId);

    if (!task) {
      return res.status(404).json({ error: 'Task not found' });
    }

    task.timeTracking.push({
      userId: req.user._id,
      timeSpent,
      date: new Date()
    });

    await task.save();

    res.json(task);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
};
```

## ML Service Implementation

### FastAPI Main Application

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from typing import List, Optional
import numpy as np
from sklearn.ensemble import RandomForestClassifier, IsolationForest
from sklearn.preprocessing import StandardScaler
import joblib
import os

app = FastAPI(title="Enterprise User Management ML Service")

# CORS middleware
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Models directory
MODEL_PATH = os.getenv("MODEL_PATH", "./models")

# Load or initialize models
ticket
