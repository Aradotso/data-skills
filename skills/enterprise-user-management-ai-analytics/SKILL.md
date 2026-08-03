---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics for risk detection, burnout analysis, and ticket classification
triggers:
  - "set up enterprise user management system"
  - "implement AI analytics for user management"
  - "configure JWT authentication for admin panel"
  - "create user dashboard with task tracking"
  - "integrate AI ticket classification system"
  - "build role-based access control system"
  - "add burnout detection and anomaly monitoring"
  - "deploy user management with AI insights"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A full-stack enterprise user management system with AI-powered analytics for managing users, tasks, and support tickets. Features JWT authentication, role-based access control, Kanban boards, time tracking, and AI capabilities including risk prediction, anomaly detection, burnout analysis, and intelligent ticket routing.

## Project Architecture

The system consists of three main components:
- **Frontend**: React.js application for user and admin interfaces
- **Backend**: Node.js REST API with JWT authentication
- **ML Service**: FastAPI-based AI analytics engine using scikit-learn and River

## Installation

### Clone and Setup

```bash
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics
```

### Backend Setup

```bash
cd backend
npm install
npm start
```

Backend runs at `http://localhost:5000`

**Required Environment Variables** (create `.env` in backend/):
```env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise-user-management
JWT_SECRET=your_jwt_secret_here
JWT_EXPIRES_IN=7d
ML_SERVICE_URL=http://localhost:8000
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt
uvicorn main:app --reload
```

ML service runs at `http://localhost:8000`

**Required Environment Variables** (create `.env` in ml-service/):
```env
MODEL_PATH=./models
LOG_LEVEL=info
```

### Frontend Setup

```bash
cd frontend
npm install
npm start
```

Frontend runs at `http://localhost:3000`

**Required Environment Variables** (create `.env` in frontend/):
```env
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
```

## Key API Endpoints

### Authentication

```javascript
// Login
POST /api/auth/login
{
  "email": "user@example.com",
  "password": "password"
}

// Register
POST /api/auth/register
{
  "name": "John Doe",
  "email": "john@example.com",
  "password": "password",
  "role": "user" // or "admin"
}
```

### User Management

```javascript
// Get all users (admin only)
GET /api/users
Headers: { Authorization: "Bearer <JWT_TOKEN>" }

// Get user by ID
GET /api/users/:id

// Update user
PUT /api/users/:id
{
  "name": "Updated Name",
  "role": "admin"
}

// Delete user (admin only)
DELETE /api/users/:id
```

### Task Management

```javascript
// Get user tasks
GET /api/tasks/user/:userId

// Create task
POST /api/tasks
{
  "title": "Complete project documentation",
  "description": "Write comprehensive docs",
  "assignedTo": "userId",
  "status": "todo", // todo, in-progress, done
  "priority": "high", // low, medium, high
  "dueDate": "2026-05-01"
}

// Update task status
PUT /api/tasks/:id/status
{
  "status": "in-progress"
}

// Track time
POST /api/tasks/:id/time
{
  "timeSpent": 3600 // seconds
}
```

### Support Tickets

```javascript
// Create ticket
POST /api/tickets
{
  "title": "System access issue",
  "description": "Cannot access admin panel",
  "priority": "high",
  "category": "technical" // technical, billing, general
}

// Get user tickets
GET /api/tickets/user/:userId

// Update ticket (admin)
PUT /api/tickets/:id
{
  "status": "resolved",
  "assignedTo": "adminUserId"
}
```

### AI Analytics Endpoints

```javascript
// Classify ticket using AI
POST /api/ml/classify-ticket
{
  "title": "Cannot login to system",
  "description": "Getting authentication error"
}
// Returns: { category: "technical", priority: "high", confidence: 0.95 }

// Detect risk
POST /api/ml/risk-detection
{
  "userId": "user123",
  "recentActivities": [...]
}
// Returns: { riskScore: 0.75, anomalies: [...] }

// Burnout analysis
POST /api/ml/burnout-analysis
{
  "userId": "user123",
  "tasksCompleted": 45,
  "avgWorkHours": 10.5,
  "missedDeadlines": 3
}
// Returns: { burnoutScore: 0.8, recommendation: "reduce workload" }

// Project insights
POST /api/ml/project-insights
{
  "projectId": "proj123",
  "tasks": [...],
  "teamSize": 5
}
// Returns: { delayProbability: 0.35, estimatedCompletion: "2026-06-15" }
```

## Frontend Integration Examples

### Authentication Hook

```javascript
// src/hooks/useAuth.js
import { useState, useEffect } from 'react';
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL;

export const useAuth = () => {
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
      const response = await axios.get(`${API_URL}/api/auth/me`);
      setUser(response.data);
    } catch (error) {
      localStorage.removeItem('token');
    } finally {
      setLoading(false);
    }
  };

  const login = async (email, password) => {
    const response = await axios.post(`${API_URL}/api/auth/login`, {
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

  return { user, loading, login, logout, isAdmin: user?.role === 'admin' };
};
```

### Task Dashboard Component

```javascript
// src/components/TaskDashboard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const TaskDashboard = ({ userId }) => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });
  const API_URL = process.env.REACT_APP_API_URL;

  useEffect(() => {
    fetchTasks();
  }, [userId]);

  const fetchTasks = async () => {
    const response = await axios.get(`${API_URL}/api/tasks/user/${userId}`);
    const grouped = {
      todo: response.data.filter(t => t.status === 'todo'),
      inProgress: response.data.filter(t => t.status === 'in-progress'),
      done: response.data.filter(t => t.status === 'done')
    };
    setTasks(grouped);
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    await axios.put(`${API_URL}/api/tasks/${taskId}/status`, {
      status: newStatus
    });
    fetchTasks();
  };

  return (
    <div className="kanban-board">
      <Column 
        title="To Do" 
        tasks={tasks.todo} 
        onMove={(id) => updateTaskStatus(id, 'in-progress')}
      />
      <Column 
        title="In Progress" 
        tasks={tasks.inProgress} 
        onMove={(id) => updateTaskStatus(id, 'done')}
      />
      <Column 
        title="Done" 
        tasks={tasks.done} 
      />
    </div>
  );
};

export default TaskDashboard;
```

### AI Ticket Classification

```javascript
// src/services/aiService.js
import axios from 'axios';

const ML_API_URL = process.env.REACT_APP_ML_API_URL;

export const classifyTicket = async (title, description) => {
  try {
    const response = await axios.post(`${ML_API_URL}/classify-ticket`, {
      title,
      description
    });
    return response.data;
  } catch (error) {
    console.error('AI classification failed:', error);
    return { category: 'general', priority: 'medium', confidence: 0 };
  }
};

export const detectUserRisk = async (userId, activities) => {
  const response = await axios.post(`${ML_API_URL}/risk-detection`, {
    userId,
    recentActivities: activities
  });
  return response.data;
};

export const analyzeBurnout = async (userData) => {
  const response = await axios.post(`${ML_API_URL}/burnout-analysis`, userData);
  return response.data;
};

// Usage in component
const CreateTicketForm = () => {
  const [formData, setFormData] = useState({ title: '', description: '' });
  const [aiSuggestion, setAiSuggestion] = useState(null);

  const handleSubmit = async (e) => {
    e.preventDefault();
    
    // Get AI classification
    const classification = await classifyTicket(
      formData.title, 
      formData.description
    );
    setAiSuggestion(classification);
    
    // Create ticket with AI suggestions
    await axios.post(`${API_URL}/api/tickets`, {
      ...formData,
      category: classification.category,
      priority: classification.priority
    });
  };

  return (
    <form onSubmit={handleSubmit}>
      {/* form fields */}
      {aiSuggestion && (
        <div className="ai-suggestion">
          Suggested Category: {aiSuggestion.category}
          Priority: {aiSuggestion.priority}
        </div>
      )}
    </form>
  );
};
```

## Backend Implementation Patterns

### JWT Authentication Middleware

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
    res.status(400).json({ error: error.message });
  }
};

exports.getUserTasks = async (req, res) => {
  try {
    const tasks = await Task.find({ 
      assignedTo: req.params.userId 
    }).sort({ createdAt: -1 });
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ error: error.message });
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
    res.status(400).json({ error: error.message });
  }
};

exports.trackTime = async (req, res) => {
  try {
    const task = await Task.findById(req.params.id);
    task.timeSpent += req.body.timeSpent;
    task.timeLog.push({
      duration: req.body.timeSpent,
      loggedBy: req.user.id,
      timestamp: Date.now()
    });
    await task.save();
    res.json(task);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
};
```

## ML Service Implementation

### FastAPI Main Application

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import List, Dict
import joblib
import numpy as np
from sklearn.feature_extraction.text import TfidfVectorizer
from river import anomaly, linear_model

app = FastAPI()

# Load or initialize models
ticket_classifier = None
anomaly_detector = anomaly.HalfSpaceTrees(n_trees=10, height=8, window_size=250)

class TicketRequest(BaseModel):
    title: str
    description: str

class RiskRequest(BaseModel):
    userId: str
    recentActivities: List[Dict]

class BurnoutRequest(BaseModel):
    userId: str
    tasksCompleted: int
    avgWorkHours: float
    missedDeadlines: int

@app.post("/classify-ticket")
async def classify_ticket(request: TicketRequest):
    """Classify support ticket using AI"""
    text = f"{request.title} {request.description}".lower()
    
    # Simple rule-based classification (replace with trained model)
    keywords = {
        'technical': ['login', 'error', 'bug', 'crash', 'system', 'access'],
        'billing': ['payment', 'invoice', 'charge', 'subscription', 'refund'],
        'general': ['question', 'help', 'how', 'information']
    }
    
    category = 'general'
    confidence = 0.5
    
    for cat, words in keywords.items():
        if any(word in text for word in words):
            category = cat
            confidence = 0.85
            break
    
    # Determine priority
    urgent_words = ['urgent', 'critical', 'asap', 'immediately', 'down']
    priority = 'high' if any(word in text for word in urgent_words) else 'medium'
    
    return {
        "category": category,
        "priority": priority,
        "confidence": confidence
    }

@app.post("/risk-detection")
async def detect_risk(request: RiskRequest):
    """Detect anomalous user behavior"""
    # Extract features from activities
    activity_count = len(request.recentActivities)
    
    if activity_count == 0:
        return {"riskScore": 0.0, "anomalies": []}
    
    # Calculate features
    failed_logins = sum(1 for a in request.recentActivities if a.get('type') == 'failed_login')
    unusual_hours = sum(1 for a in request.recentActivities if a.get('hour', 12) < 6 or a.get('hour', 12) > 22)
    
    risk_score = min(1.0, (failed_logins * 0.3 + unusual_hours * 0.2) / activity_count)
    
    anomalies = []
    if failed_logins > 3:
        anomalies.append("Multiple failed login attempts")
    if unusual_hours > 5:
        anomalies.append("Activity during unusual hours")
    
    return {
        "riskScore": round(risk_score, 2),
        "anomalies": anomalies,
        "recommendation": "Monitor closely" if risk_score > 0.7 else "Normal"
    }

@app.post("/burnout-analysis")
async def analyze_burnout(request: BurnoutRequest):
    """Analyze user burnout risk"""
    # Calculate burnout score
    workload_score = min(1.0, request.tasksCompleted / 50)  # Normalize to 50 tasks
    hours_score = min(1.0, request.avgWorkHours / 12)  # Normalize to 12 hours
    deadline_score = min(1.0, request.missedDeadlines / 5)  # Normalize to 5 misses
    
    burnout_score = (workload_score * 0.3 + hours_score * 0.5 + deadline_score * 0.2)
    
    if burnout_score > 0.7:
        recommendation = "High risk: Reduce workload immediately"
    elif burnout_score > 0.5:
        recommendation = "Moderate risk: Consider redistributing tasks"
    else:
        recommendation = "Low risk: Continue monitoring"
    
    return {
        "burnoutScore": round(burnout_score, 2),
        "recommendation": recommendation,
        "factors": {
            "workload": round(workload_score, 2),
            "hours": round(hours_score, 2),
            "deadlines": round(deadline_score, 2)
        }
    }

@app.post("/project-insights")
async def project_insights(request: Dict):
    """Predict project delays and completion"""
    project_id = request.get('projectId')
    tasks = request.get('tasks', [])
    team_size = request.get('teamSize', 1)
    
    total_tasks = len(tasks)
    completed_tasks = sum(1 for t in tasks if t.get('status') == 'done')
    overdue_tasks = sum(1 for t in tasks if t.get('isOverdue', False))
    
    completion_rate = completed_tasks / total_tasks if total_tasks > 0 else 0
    delay_probability = min(1.0, overdue_tasks / (total_tasks * 0.2))  # 20% threshold
    
    # Estimate completion
    remaining_tasks = total_tasks - completed_tasks
    estimated_days = int((remaining_tasks / team_size) * 2)  # 2 days per task per person
    
    return {
        "projectId": project_id,
        "completionRate": round(completion_rate, 2),
        "delayProbability": round(delay_probability, 2),
        "estimatedDaysToCompletion": estimated_days,
        "recommendation": "Add resources" if delay_probability > 0.6 else "On track"
    }

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "ml-analytics"}
```

## Common Patterns

### Role-Based Component Rendering

```javascript
// src/components/ProtectedRoute.jsx
import { Navigate } from 'react-router-dom';
import { useAuth } from '../hooks/useAuth';

const ProtectedRoute = ({ children, adminOnly = false }) => {
  const { user, loading } = useAuth();

  if (loading) return <div>Loading...</div>;
  if (!user) return <Navigate to="/login" />;
  if (adminOnly && user.role !== 'admin') return <Navigate to="/dashboard" />;

  return children;
};

// Usage in routes
<Route path="/admin" element={
  <ProtectedRoute adminOnly>
    <AdminDashboard />
  </ProtectedRoute>
} />
```

### Real-time Notifications

```javascript
// src/hooks/useNotifications.js
import { useEffect, useState } from 'react';
import axios from 'axios';

export const useNotifications = (userId) => {
  const [notifications, setNotifications] = useState([]);

  useEffect(() => {
    const fetchNotifications = async () => {
      const response = await axios.get(`${process.env.REACT_APP_API_URL}/api/notifications/${userId}`);
      setNotifications(response.data);
    };

    fetchNotifications();
    const interval = setInterval(fetchNotifications, 30000); // Poll every 30s

    return () => clearInterval(interval);
  }, [userId]);

  const markAsRead = async (notificationId) => {
    await axios.put(`${process.env.REACT_APP_API_URL}/api/notifications/${notificationId}/read`);
    setNotifications(prev => prev.map(n => 
      n.id === notificationId ? { ...n, read: true } : n
    ));
  };

  return { notifications, markAsRead };
};
```

## Troubleshooting

### JWT Token Issues

If you get "Invalid token" errors:
```javascript
// Check token expiration
const token = localStorage.getItem('token');
const decoded = jwt.decode(token);
if (decoded.exp * 1000 < Date.now()) {
  // Token expired, redirect to login
  localStorage.removeItem('token');
  window.location.href = '/login';
}
```

### MongoDB Connection Errors

Ensure MongoDB is running and connection string is correct:
```bash
# Check MongoDB status
sudo systemctl status mongod

# Verify connection in backend
const mongoose = require('mongoose');
mongoose.connect(process.env.MONGODB_URI, {
  useNewUrlParser: true,
  useUnifiedTopology: true
}).then(() => console.log('MongoDB connected'))
  .catch(err => console.error('MongoDB connection error:', err));
```

### CORS Issues

Configure CORS in backend:
```javascript
// backend/server.js
const cors = require('cors');

app.use(cors({
  origin: process.env.FRONTEND_URL || 'http://localhost:3000',
  credentials: true
}));
```

### ML Service Python Dependencies

If ML service fails to start:
```bash
# Create virtual environment
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install dependencies
pip install fastapi uvicorn scikit-learn river pydantic python-multipart

# Verify installation
python -c "import fastapi; import sklearn; import river; print('All packages installed')"
```

### Task Status Not Updating

Ensure proper state management:
```javascript
// Use callback pattern for state updates
setTasks(prevTasks => ({
  ...prevTasks,
  [status]: prevTasks[status].filter(t => t.id !== taskId)
}));
```

This skill provides comprehensive coverage for implementing and extending the Enterprise User Management System with AI Analytics, including authentication, task management, AI integration, and deployment patterns.
