---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for risk detection, task management, and organizational insights
triggers:
  - "set up enterprise user management system"
  - "integrate AI analytics for user management"
  - "create user dashboard with task tracking"
  - "implement AI-based ticket classification"
  - "build admin panel with analytics"
  - "add burnout detection for employees"
  - "configure JWT authentication for enterprise app"
  - "deploy user management system with ML service"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System with AI Analytics is a full-stack JavaScript/Node.js application that combines traditional user and task management with machine learning capabilities. The system provides:

- **User Management**: Role-based access control (Admin/User roles)
- **Task Tracking**: Kanban board with time tracking
- **Support Tickets**: AI-powered ticket classification and routing
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, and project delay prediction
- **Real-time Dashboard**: Performance metrics and organizational insights

The project consists of three main components:
1. **Frontend**: React.js SPA
2. **Backend**: Node.js REST API with MongoDB
3. **ML Service**: FastAPI with scikit-learn and River for online learning

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
# Clone repository
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics

# Install backend dependencies
cd backend
npm install

# Install frontend dependencies
cd ../frontend
npm install

# Install ML service dependencies
cd ../ml-service
pip install -r requirements.txt
```

### Environment Configuration

**Backend (.env)**
```bash
# backend/.env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_ums
JWT_SECRET=your_jwt_secret_key_here
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
```

**Frontend (.env)**
```bash
# frontend/.env
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_SERVICE_URL=http://localhost:8000
```

**ML Service (.env)**
```bash
# ml-service/.env
BACKEND_URL=http://localhost:5000
MODEL_PATH=./models
LOG_LEVEL=INFO
```

## Running the Application

### Start All Services

```bash
# Terminal 1 - Backend
cd backend
npm start

# Terminal 2 - ML Service
cd ml-service
uvicorn main:app --reload --port 8000

# Terminal 3 - Frontend
cd frontend
npm start
```

Access points:
- Frontend: http://localhost:3000
- Backend API: http://localhost:5000
- ML Service: http://localhost:8000
- ML API Docs: http://localhost:8000/docs

## Backend API Reference

### Authentication

```javascript
// Register new user
POST /api/auth/register
Content-Type: application/json

{
  "username": "john_doe",
  "email": "john@example.com",
  "password": "SecurePass123!",
  "role": "user"
}

// Login
POST /api/auth/login
{
  "email": "john@example.com",
  "password": "SecurePass123!"
}

// Response includes JWT token
{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "user": { "id": "...", "email": "...", "role": "user" }
}
```

### User Management (Admin Only)

```javascript
// Get all users
GET /api/users
Authorization: Bearer <token>

// Update user
PUT /api/users/:id
Authorization: Bearer <token>
{
  "role": "admin",
  "status": "active"
}

// Delete user
DELETE /api/users/:id
Authorization: Bearer <token>
```

### Task Management

```javascript
// Create task
POST /api/tasks
Authorization: Bearer <token>
{
  "title": "Implement authentication",
  "description": "Add JWT-based auth",
  "assignedTo": "user_id",
  "dueDate": "2026-05-01",
  "priority": "high",
  "status": "todo"
}

// Update task status
PATCH /api/tasks/:id/status
{
  "status": "in-progress"
}

// Get user tasks
GET /api/tasks/my-tasks
Authorization: Bearer <token>
```

### Support Tickets

```javascript
// Create ticket
POST /api/tickets
Authorization: Bearer <token>
{
  "subject": "Cannot access dashboard",
  "description": "Getting 403 error when accessing admin panel",
  "priority": "high"
}

// Get ticket with AI classification
GET /api/tickets/:id
// Response includes AI category and routing
{
  "id": "...",
  "subject": "...",
  "aiCategory": "access_control",
  "suggestedDepartment": "IT Security",
  "priority": "high"
}
```

## Frontend Integration

### API Client Setup

```javascript
// frontend/src/utils/api.js
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL;

const api = axios.create({
  baseURL: API_URL,
  headers: {
    'Content-Type': 'application/json',
  },
});

// Add token to requests
api.interceptors.request.use((config) => {
  const token = localStorage.getItem('token');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

export default api;
```

### Authentication Hook

```javascript
// frontend/src/hooks/useAuth.js
import { useState, useEffect } from 'react';
import api from '../utils/api';

export const useAuth = () => {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);

  const login = async (email, password) => {
    const response = await api.post('/auth/login', { email, password });
    localStorage.setItem('token', response.data.token);
    setUser(response.data.user);
    return response.data;
  };

  const logout = () => {
    localStorage.removeItem('token');
    setUser(null);
  };

  useEffect(() => {
    const token = localStorage.getItem('token');
    if (token) {
      api.get('/auth/me')
        .then(res => setUser(res.data))
        .catch(() => logout())
        .finally(() => setLoading(false));
    } else {
      setLoading(false);
    }
  }, []);

  return { user, loading, login, logout };
};
```

### Task Dashboard Component

```javascript
// frontend/src/components/TaskBoard.jsx
import React, { useState, useEffect } from 'react';
import api from '../utils/api';

const TaskBoard = () => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    const response = await api.get('/tasks/my-tasks');
    const grouped = response.data.reduce((acc, task) => {
      const status = task.status === 'in-progress' ? 'inProgress' : task.status;
      acc[status] = [...(acc[status] || []), task];
      return acc;
    }, { todo: [], inProgress: [], done: [] });
    setTasks(grouped);
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    await api.patch(`/tasks/${taskId}/status`, { status: newStatus });
    fetchTasks();
  };

  return (
    <div className="task-board">
      {['todo', 'inProgress', 'done'].map(status => (
        <div key={status} className="task-column">
          <h3>{status.toUpperCase()}</h3>
          {tasks[status].map(task => (
            <div key={task.id} className="task-card">
              <h4>{task.title}</h4>
              <p>{task.description}</p>
              <select 
                value={status} 
                onChange={(e) => updateTaskStatus(task.id, e.target.value)}
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
  );
};

export default TaskBoard;
```

## ML Service API

### AI-Powered Analytics Endpoints

```python
# Get AI insights for a user
GET /api/ml/user-insights/{user_id}

# Response
{
  "userId": "...",
  "riskScore": 0.75,
  "riskLevel": "high",
  "burnoutProbability": 0.45,
  "anomalyDetected": false,
  "recommendations": [
    "Consider redistributing workload",
    "Monitor task completion rate"
  ]
}
```

### Ticket Classification

```python
# Classify support ticket
POST /api/ml/classify-ticket
{
  "subject": "Password reset not working",
  "description": "I clicked reset password but didn't receive email"
}

# Response
{
  "category": "authentication",
  "priority": "medium",
  "suggestedDepartment": "IT Support",
  "confidence": 0.89
}
```

### Project Delay Prediction

```python
# Predict project delays
POST /api/ml/predict-delays
{
  "projectId": "proj_123",
  "tasksCompleted": 15,
  "totalTasks": 50,
  "daysElapsed": 30,
  "totalDays": 90
}

# Response
{
  "delayProbability": 0.62,
  "estimatedDaysDelay": 12,
  "riskFactors": [
    "Completion rate below target",
    "Multiple high-priority tasks pending"
  ]
}
```

## Common Patterns

### Protected Route Component

```javascript
// frontend/src/components/ProtectedRoute.jsx
import { Navigate } from 'react-router-dom';
import { useAuth } from '../hooks/useAuth';

const ProtectedRoute = ({ children, requireAdmin = false }) => {
  const { user, loading } = useAuth();

  if (loading) return <div>Loading...</div>;
  if (!user) return <Navigate to="/login" />;
  if (requireAdmin && user.role !== 'admin') {
    return <Navigate to="/dashboard" />;
  }

  return children;
};

export default ProtectedRoute;
```

### Admin Dashboard with Analytics

```javascript
// frontend/src/pages/AdminDashboard.jsx
import React, { useState, useEffect } from 'react';
import api from '../utils/api';
import axios from 'axios';

const AdminDashboard = () => {
  const [analytics, setAnalytics] = useState(null);
  const [users, setUsers] = useState([]);

  useEffect(() => {
    fetchDashboardData();
  }, []);

  const fetchDashboardData = async () => {
    const [usersRes, analyticsRes] = await Promise.all([
      api.get('/users'),
      api.get('/analytics/overview')
    ]);

    setUsers(usersRes.data);
    setAnalytics(analyticsRes.data);

    // Fetch AI insights for high-risk users
    const mlServiceUrl = process.env.REACT_APP_ML_SERVICE_URL;
    const insights = await Promise.all(
      usersRes.data.map(user => 
        axios.get(`${mlServiceUrl}/api/ml/user-insights/${user.id}`)
      )
    );

    const enrichedUsers = usersRes.data.map((user, idx) => ({
      ...user,
      aiInsights: insights[idx].data
    }));

    setUsers(enrichedUsers);
  };

  return (
    <div className="admin-dashboard">
      <h1>Admin Dashboard</h1>
      
      <div className="metrics">
        <div className="metric-card">
          <h3>Total Users</h3>
          <p>{analytics?.totalUsers || 0}</p>
        </div>
        <div className="metric-card">
          <h3>Active Tasks</h3>
          <p>{analytics?.activeTasks || 0}</p>
        </div>
        <div className="metric-card">
          <h3>Open Tickets</h3>
          <p>{analytics?.openTickets || 0}</p>
        </div>
      </div>

      <div className="users-table">
        <h2>User Risk Analysis</h2>
        <table>
          <thead>
            <tr>
              <th>User</th>
              <th>Role</th>
              <th>Risk Score</th>
              <th>Burnout Risk</th>
              <th>Actions</th>
            </tr>
          </thead>
          <tbody>
            {users.map(user => (
              <tr key={user.id}>
                <td>{user.username}</td>
                <td>{user.role}</td>
                <td className={`risk-${user.aiInsights?.riskLevel}`}>
                  {user.aiInsights?.riskScore.toFixed(2)}
                </td>
                <td>
                  {(user.aiInsights?.burnoutProbability * 100).toFixed(0)}%
                </td>
                <td>
                  <button onClick={() => viewUserDetails(user.id)}>
                    View
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

### Time Tracking Component

```javascript
// frontend/src/components/TimeTracker.jsx
import React, { useState, useEffect } from 'react';
import api from '../utils/api';

const TimeTracker = ({ taskId }) => {
  const [seconds, setSeconds] = useState(0);
  const [isActive, setIsActive] = useState(false);

  useEffect(() => {
    let interval = null;
    if (isActive) {
      interval = setInterval(() => {
        setSeconds(s => s + 1);
      }, 1000);
    } else if (!isActive && seconds !== 0) {
      clearInterval(interval);
    }
    return () => clearInterval(interval);
  }, [isActive, seconds]);

  const startTimer = () => setIsActive(true);
  
  const stopTimer = async () => {
    setIsActive(false);
    await api.post(`/tasks/${taskId}/time-log`, {
      duration: seconds
    });
  };

  const resetTimer = () => {
    setSeconds(0);
    setIsActive(false);
  };

  const formatTime = (totalSeconds) => {
    const hours = Math.floor(totalSeconds / 3600);
    const minutes = Math.floor((totalSeconds % 3600) / 60);
    const secs = totalSeconds % 60;
    return `${hours.toString().padStart(2, '0')}:${minutes.toString().padStart(2, '0')}:${secs.toString().padStart(2, '0')}`;
  };

  return (
    <div className="time-tracker">
      <h3>Time Tracker</h3>
      <div className="timer-display">{formatTime(seconds)}</div>
      <div className="timer-controls">
        {!isActive ? (
          <button onClick={startTimer}>Start</button>
        ) : (
          <button onClick={stopTimer}>Stop</button>
        )}
        <button onClick={resetTimer}>Reset</button>
      </div>
    </div>
  );
};

export default TimeTracker;
```

## Backend Implementation Examples

### User Model

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
    lowercase: true
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
  status: {
    type: String,
    enum: ['active', 'inactive', 'suspended'],
    default: 'active'
  },
  createdAt: {
    type: Date,
    default: Date.now
  }
});

// Hash password before saving
userSchema.pre('save', async function(next) {
  if (!this.isModified('password')) return next();
  this.password = await bcrypt.hash(this.password, 10);
  next();
});

// Compare password method
userSchema.methods.comparePassword = async function(candidatePassword) {
  return await bcrypt.compare(candidatePassword, this.password);
};

module.exports = mongoose.model('User', userSchema);
```

### Authentication Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

const JWT_SECRET = process.env.JWT_SECRET;

const authMiddleware = async (req, res, next) => {
  try {
    const token = req.header('Authorization')?.replace('Bearer ', '');
    
    if (!token) {
      return res.status(401).json({ error: 'Authentication required' });
    }

    const decoded = jwt.verify(token, JWT_SECRET);
    const user = await User.findById(decoded.userId);

    if (!user || user.status !== 'active') {
      throw new Error();
    }

    req.user = user;
    req.token = token;
    next();
  } catch (error) {
    res.status(401).json({ error: 'Invalid authentication token' });
  }
};

const adminMiddleware = (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ error: 'Admin access required' });
  }
  next();
};

module.exports = { authMiddleware, adminMiddleware };
```

### Task Controller

```javascript
// backend/controllers/taskController.js
const Task = require('../models/Task');
const axios = require('axios');

const ML_SERVICE_URL = process.env.ML_SERVICE_URL;

exports.createTask = async (req, res) => {
  try {
    const task = new Task({
      ...req.body,
      createdBy: req.user.id
    });

    await task.save();

    // Get AI insights for task assignment
    const aiResponse = await axios.post(`${ML_SERVICE_URL}/api/ml/analyze-task`, {
      taskId: task.id,
      assignedTo: task.assignedTo,
      priority: task.priority
    });

    res.status(201).json({
      task,
      aiInsights: aiResponse.data
    });
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
};

exports.getMyTasks = async (req, res) => {
  try {
    const tasks = await Task.find({
      assignedTo: req.user.id
    }).populate('createdBy', 'username email');

    res.json(tasks);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
};

exports.updateTaskStatus = async (req, res) => {
  try {
    const { id } = req.params;
    const { status } = req.body;

    const task = await Task.findById(id);

    if (!task) {
      return res.status(404).json({ error: 'Task not found' });
    }

    // Check permissions
    if (task.assignedTo.toString() !== req.user.id.toString() && 
        req.user.role !== 'admin') {
      return res.status(403).json({ error: 'Permission denied' });
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
```

## ML Service Implementation

### FastAPI Main Application

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
import joblib
import numpy as np
from typing import List, Optional

app = FastAPI(title="Enterprise UMS ML Service")

# CORS middleware
app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:3000", "http://localhost:5000"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Load ML models (these would be trained separately)
try:
    risk_model = joblib.load('./models/risk_model.pkl')
    burnout_model = joblib.load('./models/burnout_model.pkl')
    ticket_classifier = joblib.load('./models/ticket_classifier.pkl')
except FileNotFoundError:
    print("Warning: ML models not found. Using dummy predictions.")
    risk_model = None
    burnout_model = None
    ticket_classifier = None

class UserInsightRequest(BaseModel):
    userId: str
    tasksCompleted: int
    tasksInProgress: int
    avgCompletionTime: float
    ticketsRaised: int
    loginFrequency: float

class TicketClassificationRequest(BaseModel):
    subject: str
    description: str

@app.get("/")
async def root():
    return {"status": "ML Service Running", "version": "1.0.0"}

@app.get("/api/ml/user-insights/{user_id}")
async def get_user_insights(user_id: str):
    """Generate AI insights for a specific user"""
    try:
        # In production, fetch user data from database
        # For now, using mock data
        features = np.array([[10, 5, 2.5, 3, 0.8]])  # Mock features
        
        if risk_model:
            risk_score = float(risk_model.predict_proba(features)[0][1])
            burnout_prob = float(burnout_model.predict_proba(features)[0][1])
        else:
            # Dummy predictions for development
            risk_score = np.random.uniform(0.2, 0.8)
            burnout_prob = np.random.uniform(0.1, 0.6)
        
        risk_level = "high" if risk_score > 0.7 else "medium" if risk_score > 0.4 else "low"
        
        recommendations = []
        if risk_score > 0.7:
            recommendations.append("Consider redistributing workload")
        if burnout_prob > 0.5:
            recommendations.append("Monitor for signs of burnout")
            recommendations.append("Suggest workload reduction or time off")
        
        return {
            "userId": user_id,
            "riskScore": round(risk_score, 2),
            "riskLevel": risk_level,
            "burnoutProbability": round(burnout_prob, 2),
            "anomalyDetected": risk_score > 0.8,
            "recommendations": recommendations
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/classify-ticket")
async def classify_ticket(request: TicketClassificationRequest):
    """Classify support ticket using NLP"""
    try:
        text = f"{request.subject} {request.description}".lower()
        
        # Simple rule-based classification (replace with trained model)
        categories = {
            "authentication": ["login", "password", "access", "credentials"],
            "technical": ["error", "bug", "crash", "issue", "broken"],
            "feature_request": ["feature", "enhancement", "add", "improve"],
            "account": ["account", "profile", "settings", "update"]
        }
        
        category = "general"
        max_score = 0
        
        for cat, keywords in categories.items():
            score = sum(1 for kw in keywords if kw in text)
            if score > max_score:
                max_score = score
                category = cat
        
        # Determine priority based on keywords
        urgent_keywords = ["urgent", "critical", "cannot", "blocked", "down"]
        priority = "high" if any(kw in text for kw in urgent_keywords) else "medium"
        
        # Route to appropriate department
        department_mapping = {
            "authentication": "IT Security",
            "technical": "Technical Support",
            "feature_request": "Product Team",
            "account": "Customer Service",
            "general": "General Support"
        }
        
        return {
            "category": category,
            "priority": priority,
            "suggestedDepartment": department_mapping.get(category, "General Support"),
            "confidence": 0.85 if max_score > 0 else 0.5
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/predict-delays")
async def predict_project_delays(
    projectId: str,
    tasksCompleted: int,
    totalTasks: int,
    daysElapsed: int,
    totalDays: int
):
    """Predict project delay probability"""
    try:
        completion_rate = tasksCompleted / totalTasks if totalTasks > 0 else 0
        expected_completion = daysElapsed / totalDays if totalDays > 0 else 0
        
        # Calculate delay probability
        if completion_rate < expected_completion * 0.8:
            delay_prob = 0.8
            estimated_delay = int((totalDays - daysElapsed) * 0.3)
            risk_factors = [
                "Completion rate significantly below target",
                "High risk of missing deadline"
            ]
        elif completion_rate < expected_completion:
            delay_prob = 0.5
            estimated_delay = int((totalDays - daysElapsed) * 0.15)
            risk_factors = ["Completion rate below target"]
        else:
            delay_prob = 0.2
            estimated_delay = 0
            risk_factors = []
        
        return {
            "projectId": projectId,
            "delayProbability": delay_prob,
            "estimatedDaysDelay": estimated_delay,
            "riskFactors": risk_factors,
            "completionRate": completion_rate,
            "onTrack": delay_prob < 0.3
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

## Configuration

### Database Setup

```javascript
// backend/config/database.js
const mongoose = require('mongoose');

const connectDB = async () => {
  try {
    const conn = await mongoose.connect(process.env.MONGODB_URI, {
      useNewUrlParser: true,
      useUnifiedTopology: true,
    });

    console.log(`MongoDB Connected: ${conn.connection.host}`);
  } catch (error) {
    console.error(`Error: ${error.message}`);
    process.exit(1);
  }
};

module.exports = connectDB;
```

### Server Setup

```javascript
// backend/server.js
require('dotenv').config();
const express = require('express');
const cors = require('cors');
const connectDB = require('./config/database');

const authRoutes = require('./routes/auth');
const userRoutes = require('./routes/users');
const taskRoutes = require('./routes/tasks');
const ticketRoutes = require('./routes/tickets');

const app = express();

// Connect to database
connectDB();

// Middleware
app.use(cors());
app.use(express.json());

// Routes
app.use('/api/auth', authRoutes);
app.use('/api/users', userRoutes);
app.use('/api/tasks', taskRoutes);
app.use('/api/tickets', ticketRoutes);

// Error handling middleware
app.use((err, req, res, next) => {
  console.error(err.stack);
  res.status(500).json({ error: 'Something went wrong!' });
});

const PORT = process.env.PORT || 5000;

app.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});
```

## Troubleshooting

### JWT Token Issues

```javascript
// If getting "Invalid token" errors, check token format
const token = localStorage.getItem('token');
console.log('Token format:', token?.substring(0, 20));

// Verify token expiration
const decoded = jwt.decode(token);
console.log('Token expires at:', new Date(decoded.exp * 1000));

// Clear expired tokens
if (Date.now() >= decoded.exp * 1000) {
  localStorage.removeItem('token');
  window.location.href = '/login';
}
```

### MongoDB Connection Issues

```bash
# Check MongoDB is running
mongod --version

# Start MongoDB service
# macOS
brew services start mongodb-community

# Linux
sudo systemctl start mongod

# Windows
net start MongoDB
```

### ML Service Not Responding

```python
# Check if models directory exists
import os
if not os.path.exists('./models'):
    os.makedirs('./models')
    print("Created models directory")

# Test ML service independently
curl http://localhost:8000/
# Should return: {"status":"ML Service Running","version":"1.0.0"}

# Check Python dependencies
pip list | grep -E "(fastapi|scikit-learn|river)"
```

### CORS Errors

```javascript
// Backend - ensure CORS is properly configured
const cors = require('cors');
app.use(cors({
  origin: ['http://localhost:3000', 'http://localhost:5000'],
  credentials: true
}));

// Frontend - check API URL
console.log('API URL:', process.env.REACT_APP_API_URL);
```

### Task Status Not Updating

```javascript
// Verify task ownership
const task = await Task.findById(taskId);
console.log('Task assigned to:', task.assignedTo);
console.log('Current user:', req.user.id);

// Check status enum values
const validStatuses = ['todo', 'in-progress', 'done'];
if (!validStatuses.includes(status)) {
  throw new Error('Invalid status value');
}
```

### Performance
