---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics, risk detection, and task management for enterprise organizations
triggers:
  - "set up enterprise user management system"
  - "implement AI analytics for user management"
  - "create user dashboard with task tracking"
  - "add AI-based ticket classification"
  - "build admin panel for user management"
  - "integrate burnout detection and risk prediction"
  - "configure JWT authentication for enterprise app"
  - "deploy user management system with AI features"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This is a full-stack web application for managing users, tasks, and support tickets with integrated AI analytics. It provides risk detection, anomaly detection, burnout analysis, and predictive insights to improve organizational productivity and decision-making.

## What It Does

- **User Management**: Role-based access control, user CRUD operations, authentication with JWT
- **Task Management**: Kanban board (To Do, In Progress, Done), time tracking, task assignment
- **Support System**: Ticket creation, AI-based classification and routing
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, project delay prediction
- **Admin Dashboard**: Organization analytics, audit logs, user monitoring
- **User Dashboard**: Personal task overview, performance insights, notifications

## Installation

### Prerequisites

- Node.js (v14+)
- Python (v3.8+)
- MongoDB

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

Create `.env` file in backend directory:

```env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
JWT_SECRET=your_jwt_secret_here
ML_SERVICE_URL=http://localhost:8000
```

Start backend:

```bash
npm start
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt
```

Create `.env` file in ml-service directory:

```env
BACKEND_URL=http://localhost:5000
MODEL_PATH=./models
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

Create `.env` file in frontend directory:

```env
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_URL=http://localhost:8000
```

Start frontend:

```bash
npm start
```

## Key API Endpoints

### Authentication

```javascript
// Login
POST /api/auth/login
{
  "email": "user@example.com",
  "password": "password123"
}

// Register
POST /api/auth/register
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
Headers: { Authorization: "Bearer <JWT_TOKEN>" }

// Create user
POST /api/users
{
  "name": "Jane Smith",
  "email": "jane@example.com",
  "role": "user",
  "department": "Engineering"
}

// Update user
PUT /api/users/:id
{
  "role": "admin",
  "status": "active"
}

// Delete user
DELETE /api/users/:id
```

### Task Management

```javascript
// Get user tasks
GET /api/tasks/user/:userId

// Create task
POST /api/tasks
{
  "title": "Implement feature X",
  "description": "Details here",
  "assignedTo": "userId123",
  "priority": "high",
  "dueDate": "2026-05-01",
  "status": "todo"
}

// Update task status
PATCH /api/tasks/:taskId/status
{
  "status": "in-progress"
}

// Track time
POST /api/tasks/:taskId/time
{
  "duration": 3600  // seconds
}
```

### Support Tickets

```javascript
// Create ticket
POST /api/tickets
{
  "subject": "Login issue",
  "description": "Cannot access dashboard",
  "priority": "high",
  "category": "technical"
}

// Get tickets
GET /api/tickets

// Update ticket
PATCH /api/tickets/:ticketId
{
  "status": "resolved",
  "assignedTo": "adminId123"
}
```

### AI Analytics Endpoints

```javascript
// Risk prediction
POST /api/ml/predict-risk
{
  "userId": "user123",
  "recentActivity": [...],
  "loginPattern": [...]
}

// Burnout detection
POST /api/ml/detect-burnout
{
  "userId": "user123",
  "workHours": 55,
  "tasksCompleted": 12,
  "overdueTasksCount": 3
}

// Anomaly detection
POST /api/ml/detect-anomaly
{
  "userId": "user123",
  "loginTime": "2026-04-15T03:30:00Z",
  "location": "192.168.1.100"
}

// Ticket classification
POST /api/ml/classify-ticket
{
  "subject": "Password reset needed",
  "description": "I forgot my password"
}
```

## Frontend Usage Patterns

### Authentication with JWT

```javascript
// src/services/authService.js
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL;

export const login = async (email, password) => {
  const response = await axios.post(`${API_URL}/auth/login`, {
    email,
    password
  });
  
  if (response.data.token) {
    localStorage.setItem('token', response.data.token);
    localStorage.setItem('user', JSON.stringify(response.data.user));
  }
  
  return response.data;
};

export const getAuthHeader = () => {
  const token = localStorage.getItem('token');
  return token ? { Authorization: `Bearer ${token}` } : {};
};

export const logout = () => {
  localStorage.removeItem('token');
  localStorage.removeItem('user');
};
```

### Task Management Component

```javascript
// src/components/TaskBoard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import { getAuthHeader } from '../services/authService';

const TaskBoard = () => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });
  const API_URL = process.env.REACT_APP_API_URL;

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    try {
      const response = await axios.get(`${API_URL}/tasks/user/me`, {
        headers: getAuthHeader()
      });
      
      const grouped = {
        todo: response.data.filter(t => t.status === 'todo'),
        inProgress: response.data.filter(t => t.status === 'in-progress'),
        done: response.data.filter(t => t.status === 'done')
      };
      
      setTasks(grouped);
    } catch (error) {
      console.error('Error fetching tasks:', error);
    }
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    try {
      await axios.patch(
        `${API_URL}/tasks/${taskId}/status`,
        { status: newStatus },
        { headers: getAuthHeader() }
      );
      fetchTasks();
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  return (
    <div className="task-board">
      <div className="column">
        <h3>To Do</h3>
        {tasks.todo.map(task => (
          <TaskCard 
            key={task._id} 
            task={task} 
            onStatusChange={updateTaskStatus}
          />
        ))}
      </div>
      <div className="column">
        <h3>In Progress</h3>
        {tasks.inProgress.map(task => (
          <TaskCard 
            key={task._id} 
            task={task} 
            onStatusChange={updateTaskStatus}
          />
        ))}
      </div>
      <div className="column">
        <h3>Done</h3>
        {tasks.done.map(task => (
          <TaskCard 
            key={task._id} 
            task={task} 
            onStatusChange={updateTaskStatus}
          />
        ))}
      </div>
    </div>
  );
};
```

### AI Analytics Integration

```javascript
// src/services/aiService.js
import axios from 'axios';
import { getAuthHeader } from './authService';

const ML_URL = process.env.REACT_APP_ML_URL;

export const predictRisk = async (userId, activityData) => {
  try {
    const response = await axios.post(
      `${ML_URL}/predict-risk`,
      {
        userId,
        recentActivity: activityData
      },
      { headers: getAuthHeader() }
    );
    return response.data;
  } catch (error) {
    console.error('Risk prediction error:', error);
    throw error;
  }
};

export const detectBurnout = async (userMetrics) => {
  try {
    const response = await axios.post(
      `${ML_URL}/detect-burnout`,
      userMetrics,
      { headers: getAuthHeader() }
    );
    return response.data;
  } catch (error) {
    console.error('Burnout detection error:', error);
    throw error;
  }
};

export const classifyTicket = async (ticketData) => {
  try {
    const response = await axios.post(
      `${ML_URL}/classify-ticket`,
      ticketData,
      { headers: getAuthHeader() }
    );
    return response.data;
  } catch (error) {
    console.error('Ticket classification error:', error);
    throw error;
  }
};
```

## Backend Implementation Examples

### User Controller

```javascript
// backend/controllers/userController.js
const User = require('../models/User');
const jwt = require('jsonwebtoken');

// Get all users (Admin only)
exports.getAllUsers = async (req, res) => {
  try {
    const users = await User.find().select('-password');
    res.json(users);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
};

// Create user
exports.createUser = async (req, res) => {
  try {
    const { name, email, password, role, department } = req.body;
    
    const existingUser = await User.findOne({ email });
    if (existingUser) {
      return res.status(400).json({ message: 'User already exists' });
    }

    const user = new User({
      name,
      email,
      password,
      role,
      department
    });

    await user.save();
    res.status(201).json({ message: 'User created successfully', user });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
};

// Update user
exports.updateUser = async (req, res) => {
  try {
    const { id } = req.params;
    const updates = req.body;

    const user = await User.findByIdAndUpdate(
      id,
      updates,
      { new: true, runValidators: true }
    ).select('-password');

    if (!user) {
      return res.status(404).json({ message: 'User not found' });
    }

    res.json({ message: 'User updated successfully', user });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
};

// Delete user
exports.deleteUser = async (req, res) => {
  try {
    const { id } = req.params;
    const user = await User.findByIdAndDelete(id);

    if (!user) {
      return res.status(404).json({ message: 'User not found' });
    }

    res.json({ message: 'User deleted successfully' });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
};
```

### Task Controller

```javascript
// backend/controllers/taskController.js
const Task = require('../models/Task');

// Create task
exports.createTask = async (req, res) => {
  try {
    const { title, description, assignedTo, priority, dueDate, status } = req.body;
    
    const task = new Task({
      title,
      description,
      assignedTo,
      priority: priority || 'medium',
      dueDate,
      status: status || 'todo',
      createdBy: req.user.id
    });

    await task.save();
    res.status(201).json({ message: 'Task created successfully', task });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
};

// Get user tasks
exports.getUserTasks = async (req, res) => {
  try {
    const userId = req.params.userId === 'me' ? req.user.id : req.params.userId;
    
    const tasks = await Task.find({ assignedTo: userId })
      .populate('assignedTo', 'name email')
      .populate('createdBy', 'name email')
      .sort({ createdAt: -1 });

    res.json(tasks);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
};

// Update task status
exports.updateTaskStatus = async (req, res) => {
  try {
    const { taskId } = req.params;
    const { status } = req.body;

    const task = await Task.findByIdAndUpdate(
      taskId,
      { status, updatedAt: Date.now() },
      { new: true }
    );

    if (!task) {
      return res.status(404).json({ message: 'Task not found' });
    }

    res.json({ message: 'Task status updated', task });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
};

// Track time
exports.trackTime = async (req, res) => {
  try {
    const { taskId } = req.params;
    const { duration } = req.body;

    const task = await Task.findById(taskId);
    if (!task) {
      return res.status(404).json({ message: 'Task not found' });
    }

    task.timeSpent = (task.timeSpent || 0) + duration;
    await task.save();

    res.json({ message: 'Time tracked successfully', task });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
};
```

### Authentication Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

exports.authenticate = async (req, res, next) => {
  try {
    const token = req.header('Authorization')?.replace('Bearer ', '');
    
    if (!token) {
      return res.status(401).json({ message: 'No authentication token' });
    }

    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    const user = await User.findById(decoded.id).select('-password');

    if (!user) {
      return res.status(401).json({ message: 'User not found' });
    }

    req.user = user;
    next();
  } catch (error) {
    res.status(401).json({ message: 'Invalid authentication token' });
  }
};

exports.authorizeAdmin = (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ message: 'Admin access required' });
  }
  next();
};
```

## ML Service Implementation

### FastAPI ML Service

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import List, Optional
import numpy as np
from sklearn.ensemble import RandomForestClassifier
import joblib
import os

app = FastAPI()

# Load or initialize models
MODEL_PATH = os.getenv('MODEL_PATH', './models')

class RiskPredictionRequest(BaseModel):
    userId: str
    recentActivity: List[dict]
    loginPattern: Optional[List[dict]] = []

class BurnoutDetectionRequest(BaseModel):
    userId: str
    workHours: float
    tasksCompleted: int
    overdueTasksCount: int
    avgTaskCompletionTime: Optional[float] = 0

class TicketClassificationRequest(BaseModel):
    subject: str
    description: str

@app.post("/predict-risk")
async def predict_risk(request: RiskPredictionRequest):
    try:
        # Extract features from activity data
        features = extract_risk_features(request.recentActivity, request.loginPattern)
        
        # Load risk prediction model
        model = load_model('risk_classifier')
        
        # Predict risk level
        risk_score = model.predict_proba([features])[0][1]
        risk_level = "high" if risk_score > 0.7 else "medium" if risk_score > 0.4 else "low"
        
        return {
            "userId": request.userId,
            "riskScore": float(risk_score),
            "riskLevel": risk_level,
            "factors": analyze_risk_factors(features)
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/detect-burnout")
async def detect_burnout(request: BurnoutDetectionRequest):
    try:
        # Calculate burnout score based on metrics
        burnout_score = calculate_burnout_score(
            request.workHours,
            request.tasksCompleted,
            request.overdueTasksCount,
            request.avgTaskCompletionTime
        )
        
        burnout_level = "high" if burnout_score > 0.7 else "medium" if burnout_score > 0.4 else "low"
        
        recommendations = generate_burnout_recommendations(burnout_score, request)
        
        return {
            "userId": request.userId,
            "burnoutScore": float(burnout_score),
            "burnoutLevel": burnout_level,
            "recommendations": recommendations
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/classify-ticket")
async def classify_ticket(request: TicketClassificationRequest):
    try:
        # Combine subject and description
        text = f"{request.subject} {request.description}"
        
        # Classify ticket
        category = classify_support_ticket(text)
        priority = determine_priority(text, category)
        suggested_assignee = suggest_assignee(category)
        
        return {
            "category": category,
            "priority": priority,
            "suggestedAssignee": suggested_assignee,
            "confidence": 0.85
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/detect-anomaly")
async def detect_anomaly(request: dict):
    try:
        features = extract_anomaly_features(request)
        model = load_model('anomaly_detector')
        
        is_anomaly = model.predict([features])[0]
        anomaly_score = model.decision_function([features])[0]
        
        return {
            "isAnomaly": bool(is_anomaly),
            "anomalyScore": float(anomaly_score),
            "details": analyze_anomaly(request, is_anomaly)
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

# Helper functions
def extract_risk_features(activity, login_pattern):
    # Extract numerical features from activity data
    features = [
        len(activity),
        len(login_pattern),
        calculate_activity_variance(activity),
        calculate_login_irregularity(login_pattern)
    ]
    return features

def calculate_burnout_score(work_hours, tasks_completed, overdue_tasks, avg_completion_time):
    # Normalize and weight different factors
    hour_score = min(work_hours / 60, 1.0) * 0.4
    overdue_score = min(overdue_tasks / 10, 1.0) * 0.3
    completion_score = min(avg_completion_time / 10, 1.0) * 0.3
    
    return hour_score + overdue_score + completion_score

def classify_support_ticket(text):
    # Simple keyword-based classification (can be replaced with ML model)
    keywords = {
        'technical': ['login', 'password', 'error', 'bug', 'crash'],
        'access': ['permission', 'access', 'role', 'authorization'],
        'feature': ['feature', 'enhancement', 'improvement'],
        'general': []
    }
    
    text_lower = text.lower()
    for category, words in keywords.items():
        if any(word in text_lower for word in words):
            return category
    
    return 'general'

def determine_priority(text, category):
    urgent_keywords = ['urgent', 'critical', 'emergency', 'asap', 'immediately']
    text_lower = text.lower()
    
    if any(word in text_lower for word in urgent_keywords):
        return 'high'
    elif category == 'technical':
        return 'medium'
    else:
        return 'low'

def suggest_assignee(category):
    assignee_map = {
        'technical': 'tech-support-team',
        'access': 'admin-team',
        'feature': 'product-team',
        'general': 'support-team'
    }
    return assignee_map.get(category, 'support-team')

def load_model(model_name):
    # Load pre-trained model or return dummy model
    model_file = os.path.join(MODEL_PATH, f'{model_name}.pkl')
    if os.path.exists(model_file):
        return joblib.load(model_file)
    else:
        # Return a simple classifier for demonstration
        return RandomForestClassifier(n_estimators=10)

def analyze_risk_factors(features):
    return ["Unusual login pattern", "High activity variance"]

def generate_burnout_recommendations(score, request):
    recommendations = []
    if request.workHours > 50:
        recommendations.append("Consider reducing work hours")
    if request.overdueTasksCount > 5:
        recommendations.append("Prioritize overdue tasks")
    return recommendations

def extract_anomaly_features(request):
    return [0, 0, 0, 0]  # Placeholder

def analyze_anomaly(request, is_anomaly):
    return {"message": "Unusual activity detected" if is_anomaly else "Normal activity"}

def calculate_activity_variance(activity):
    return 0.5  # Placeholder

def calculate_login_irregularity(login_pattern):
    return 0.3  # Placeholder
```

## Configuration

### MongoDB Models

```javascript
// backend/models/User.js
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');

const userSchema = new mongoose.Schema({
  name: { type: String, required: true },
  email: { type: String, required: true, unique: true },
  password: { type: String, required: true },
  role: { type: String, enum: ['admin', 'user'], default: 'user' },
  department: String,
  status: { type: String, enum: ['active', 'inactive'], default: 'active' },
  createdAt: { type: Date, default: Date.now },
  lastLogin: Date
});

userSchema.pre('save', async function(next) {
  if (this.isModified('password')) {
    this.password = await bcrypt.hash(this.password, 10);
  }
  next();
});

userSchema.methods.comparePassword = async function(candidatePassword) {
  return await bcrypt.compare(candidatePassword, this.password);
};

module.exports = mongoose.model('User', userSchema);
```

```javascript
// backend/models/Task.js
const mongoose = require('mongoose');

const taskSchema = new mongoose.Schema({
  title: { type: String, required: true },
  description: String,
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
  createdBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  status: { 
    type: String, 
    enum: ['todo', 'in-progress', 'done'], 
    default: 'todo' 
  },
  priority: { 
    type: String, 
    enum: ['low', 'medium', 'high'], 
    default: 'medium' 
  },
  dueDate: Date,
  timeSpent: { type: Number, default: 0 },
  createdAt: { type: Date, default: Date.now },
  updatedAt: Date
});

module.exports = mongoose.model('Task', taskSchema);
```

```javascript
// backend/models/Ticket.js
const mongoose = require('mongoose');

const ticketSchema = new mongoose.Schema({
  subject: { type: String, required: true },
  description: { type: String, required: true },
  category: String,
  priority: { 
    type: String, 
    enum: ['low', 'medium', 'high'], 
    default: 'medium' 
  },
  status: { 
    type: String, 
    enum: ['open', 'in-progress', 'resolved', 'closed'], 
    default: 'open' 
  },
  createdBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  createdAt: { type: Date, default: Date.now },
  resolvedAt: Date
});

module.exports = mongoose.model('Ticket', ticketSchema);
```

## Common Patterns

### Admin Dashboard with Analytics

```javascript
// src/pages/AdminDashboard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import { getAuthHeader } from '../services/authService';

const AdminDashboard = () => {
  const [analytics, setAnalytics] = useState(null);
  const [users, setUsers] = useState([]);
  const [riskAlerts, setRiskAlerts] = useState([]);
  const API_URL = process.env.REACT_APP_API_URL;

  useEffect(() => {
    fetchDashboardData();
  }, []);

  const fetchDashboardData = async () => {
    try {
      const [analyticsRes, usersRes, alertsRes] = await Promise.all([
        axios.get(`${API_URL}/analytics/overview`, { headers: getAuthHeader() }),
        axios.get(`${API_URL}/users`, { headers: getAuthHeader() }),
        axios.get(`${API_URL}/analytics/risk-alerts`, { headers: getAuthHeader() })
      ]);

      setAnalytics(analyticsRes.data);
      setUsers(usersRes.data);
      setRiskAlerts(alertsRes.data);
    } catch (error) {
      console.error('Error fetching dashboard data:', error);
    }
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
          <div className="card">
            <h3>Risk Alerts</h3>
            <p>{riskAlerts.length}</p>
          </div>
        </div>
      )}

      <div className="user-list">
        <h2>Users</h2>
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
                  <button onClick={() => handleEditUser(user._id)}>Edit</button>
                  <button onClick={() => handleDeleteUser(user._id)}>Delete</button>
                </td>
              </tr>
            ))}
          </tbody>
        </table>
      </div>
    </div>
  );
};
```

### Time Tracking Component

```javascript
// src/components/TimeTracker.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import { getAuthHeader } from '../services/authService';

const TimeTracker = ({ taskId }) => {
  const [isTracking, setIsTracking] = useState(false);
  const [seconds, setSeconds] = useState(0);
  const API_URL = process.env.REACT_APP_API_URL;

  useEffect(() => {
    let interval = null;
    if (isTracking) {
      interval = setInterval(() => {
        setSeconds(s => s + 1);
      }, 1000);
    }
    return () => clearInterval(interval);
  }, [isTracking]);

  const startTracking = () => {
    setIsTracking(true);
    setSeconds(0);
  };

  const stopTracking = async () => {
    setIsTracking(false);
    
    try {
      await axios.post(
        `${API_URL}/tasks/${taskId}/time`,
        { duration: seconds },
        { headers: getAuthHeader() }
      );
      alert('Time logged successfully');
    } catch (error) {
      console.error('Error logging time:', error);
    }
  };

  const formatTime = (totalSeconds) => {
    const hours = Math.floor(totalSeconds / 3600);
    const minutes = Math.floor((totalSeconds % 3600) / 60);
    const secs = totalSeconds % 60;
    return `${hours.toString().padStart(2, '0')}:${minutes.
