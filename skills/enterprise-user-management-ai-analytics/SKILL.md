---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics, risk detection, and task automation using React, Node.js, and FastAPI ML services
triggers:
  - "set up enterprise user management system"
  - "implement AI analytics for user management"
  - "create user task tracking with ML insights"
  - "build admin dashboard with anomaly detection"
  - "integrate AI ticket classification system"
  - "add burnout detection to user management"
  - "configure JWT authentication for enterprise app"
  - "deploy user management system with AI features"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A comprehensive full-stack user management system that combines traditional CRUD operations with AI-powered analytics including risk prediction, anomaly detection, burnout analysis, and intelligent ticket routing. Built with React frontend, Node.js backend, FastAPI ML service, and MongoDB database.

## What This Project Does

- **User Management**: Complete CRUD operations for users with role-based access control (Admin/User)
- **Task Tracking**: Kanban board (To Do → In Progress → Done) with time tracking
- **Support Tickets**: Raise, track, and manage support requests with AI-based classification
- **AI Analytics**: Risk detection, anomaly detection, burnout prediction, project delay forecasting
- **Authentication**: JWT-based secure authentication and authorization
- **Audit Logging**: Track all system activities for compliance and security

## Installation

### Prerequisites

```bash
# Node.js 14+ and npm
node --version
npm --version

# Python 3.8+ and pip
python --version
pip --version

# MongoDB running locally or connection URI
```

### Full System Setup

```bash
# Clone repository
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics

# Install and run backend
cd backend
npm install
npm start
# Runs on http://localhost:5000

# Install and run ML service (new terminal)
cd ../ml-service
pip install -r requirements.txt
uvicorn main:app --reload
# Runs on http://localhost:8000

# Install and run frontend (new terminal)
cd ../frontend
npm install
npm start
# Runs on http://localhost:3000
```

### Environment Configuration

**Backend (.env)**
```bash
# backend/.env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/user-management
JWT_SECRET=your_jwt_secret_key_here
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
```

**ML Service (.env)**
```bash
# ml-service/.env
PORT=8000
MONGODB_URI=mongodb://localhost:27017/user-management
MODEL_PATH=./models
LOG_LEVEL=INFO
```

**Frontend (.env)**
```bash
# frontend/.env
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_API_URL=http://localhost:8000
```

## Key APIs and Usage

### Authentication API

**User Registration**
```javascript
// POST /api/auth/register
const registerUser = async (userData) => {
  const response = await fetch('http://localhost:5000/api/auth/register', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      name: userData.name,
      email: userData.email,
      password: userData.password,
      role: 'user' // or 'admin'
    })
  });
  return await response.json();
};
```

**User Login**
```javascript
// POST /api/auth/login
const loginUser = async (email, password) => {
  const response = await fetch('http://localhost:5000/api/auth/login', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({ email, password })
  });
  const data = await response.json();
  // Store token
  localStorage.setItem('token', data.token);
  return data;
};
```

### User Management API (Admin)

**Get All Users**
```javascript
// GET /api/users
const getAllUsers = async () => {
  const token = localStorage.getItem('token');
  const response = await fetch('http://localhost:5000/api/users', {
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return await response.json();
};
```

**Update User**
```javascript
// PUT /api/users/:id
const updateUser = async (userId, updates) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`http://localhost:5000/api/users/${userId}`, {
    method: 'PUT',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify(updates)
  });
  return await response.json();
};
```

**Delete User**
```javascript
// DELETE /api/users/:id
const deleteUser = async (userId) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`http://localhost:5000/api/users/${userId}`, {
    method: 'DELETE',
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return await response.json();
};
```

### Task Management API

**Create Task**
```javascript
// POST /api/tasks
const createTask = async (taskData) => {
  const token = localStorage.getItem('token');
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
      priority: 'medium', // low, medium, high
      status: 'todo', // todo, inprogress, done
      dueDate: taskData.dueDate
    })
  });
  return await response.json();
};
```

**Update Task Status**
```javascript
// PATCH /api/tasks/:id/status
const updateTaskStatus = async (taskId, newStatus) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`http://localhost:5000/api/tasks/${taskId}/status`, {
    method: 'PATCH',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify({ status: newStatus })
  });
  return await response.json();
};
```

**Track Time on Task**
```javascript
// POST /api/tasks/:id/time
const trackTime = async (taskId, timeSpent) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`http://localhost:5000/api/tasks/${taskId}/time`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify({ 
      timeSpent: timeSpent, // in minutes
      timestamp: new Date().toISOString()
    })
  });
  return await response.json();
};
```

### Support Ticket API

**Create Ticket**
```javascript
// POST /api/tickets
const createTicket = async (ticketData) => {
  const token = localStorage.getItem('token');
  const response = await fetch('http://localhost:5000/api/tickets', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify({
      title: ticketData.title,
      description: ticketData.description,
      category: ticketData.category, // technical, billing, general
      priority: 'medium'
    })
  });
  return await response.json();
};
```

**Get User Tickets**
```javascript
// GET /api/tickets/my-tickets
const getMyTickets = async () => {
  const token = localStorage.getItem('token');
  const response = await fetch('http://localhost:5000/api/tickets/my-tickets', {
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return await response.json();
};
```

### AI/ML Analytics API

**Risk Prediction**
```javascript
// POST /api/ml/predict-risk
const predictUserRisk = async (userId) => {
  const token = localStorage.getItem('token');
  const response = await fetch('http://localhost:8000/predict-risk', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify({
      user_id: userId,
      task_completion_rate: 0.85,
      average_task_time: 120,
      overdue_tasks: 2,
      login_frequency: 5.2
    })
  });
  return await response.json();
  // Returns: { risk_score: 0.23, risk_level: "low", factors: [...] }
};
```

**Anomaly Detection**
```javascript
// POST /api/ml/detect-anomaly
const detectAnomaly = async (userActivity) => {
  const response = await fetch('http://localhost:8000/detect-anomaly', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      user_id: userActivity.userId,
      login_time: userActivity.loginTime,
      ip_address: userActivity.ipAddress,
      location: userActivity.location,
      device_type: userActivity.deviceType,
      failed_login_attempts: userActivity.failedAttempts
    })
  });
  return await response.json();
  // Returns: { is_anomaly: false, confidence: 0.92, anomaly_type: null }
};
```

**Burnout Detection**
```javascript
// POST /api/ml/detect-burnout
const detectBurnout = async (userId) => {
  const response = await fetch('http://localhost:8000/detect-burnout', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      user_id: userId,
      weekly_hours: 52,
      task_count: 18,
      completion_rate: 0.75,
      stress_indicators: {
        missed_deadlines: 3,
        late_submissions: 5,
        overtime_hours: 12
      }
    })
  });
  return await response.json();
  // Returns: { burnout_risk: "high", score: 0.78, recommendations: [...] }
};
```

**AI Ticket Classification**
```javascript
// POST /api/ml/classify-ticket
const classifyTicket = async (ticketContent) => {
  const response = await fetch('http://localhost:8000/classify-ticket', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      title: ticketContent.title,
      description: ticketContent.description
    })
  });
  return await response.json();
  // Returns: { category: "technical", priority: "high", suggested_assignee: "team_lead" }
};
```

**Project Delay Prediction**
```javascript
// POST /api/ml/predict-delay
const predictProjectDelay = async (projectData) => {
  const response = await fetch('http://localhost:8000/predict-delay', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      project_id: projectData.id,
      total_tasks: projectData.totalTasks,
      completed_tasks: projectData.completedTasks,
      team_size: projectData.teamSize,
      average_velocity: projectData.velocity,
      deadline: projectData.deadline
    })
  });
  return await response.json();
  // Returns: { delay_probability: 0.65, estimated_delay_days: 5, risk_factors: [...] }
};
```

## React Component Examples

### User Dashboard Component
```javascript
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const UserDashboard = () => {
  const [tasks, setTasks] = useState([]);
  const [tickets, setTickets] = useState([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchDashboardData();
  }, []);

  const fetchDashboardData = async () => {
    const token = localStorage.getItem('token');
    const config = {
      headers: { Authorization: `Bearer ${token}` }
    };

    try {
      const [tasksRes, ticketsRes] = await Promise.all([
        axios.get(`${process.env.REACT_APP_API_URL}/tasks/my-tasks`, config),
        axios.get(`${process.env.REACT_APP_API_URL}/tickets/my-tickets`, config)
      ]);
      
      setTasks(tasksRes.data);
      setTickets(ticketsRes.data);
      setLoading(false);
    } catch (error) {
      console.error('Error fetching dashboard data:', error);
      setLoading(false);
    }
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    const token = localStorage.getItem('token');
    try {
      await axios.patch(
        `${process.env.REACT_APP_API_URL}/tasks/${taskId}/status`,
        { status: newStatus },
        { headers: { Authorization: `Bearer ${token}` } }
      );
      fetchDashboardData();
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  if (loading) return <div>Loading...</div>;

  return (
    <div className="dashboard">
      <h1>My Dashboard</h1>
      
      <section className="tasks-section">
        <h2>My Tasks</h2>
        <div className="kanban-board">
          <div className="column">
            <h3>To Do</h3>
            {tasks.filter(t => t.status === 'todo').map(task => (
              <div key={task._id} className="task-card">
                <h4>{task.title}</h4>
                <p>{task.description}</p>
                <button onClick={() => updateTaskStatus(task._id, 'inprogress')}>
                  Start
                </button>
              </div>
            ))}
          </div>
          
          <div className="column">
            <h3>In Progress</h3>
            {tasks.filter(t => t.status === 'inprogress').map(task => (
              <div key={task._id} className="task-card">
                <h4>{task.title}</h4>
                <p>{task.description}</p>
                <button onClick={() => updateTaskStatus(task._id, 'done')}>
                  Complete
                </button>
              </div>
            ))}
          </div>
          
          <div className="column">
            <h3>Done</h3>
            {tasks.filter(t => t.status === 'done').map(task => (
              <div key={task._id} className="task-card completed">
                <h4>{task.title}</h4>
                <p>{task.description}</p>
              </div>
            ))}
          </div>
        </div>
      </section>

      <section className="tickets-section">
        <h2>My Tickets</h2>
        {tickets.map(ticket => (
          <div key={ticket._id} className="ticket-card">
            <h4>{ticket.title}</h4>
            <span className={`status ${ticket.status}`}>{ticket.status}</span>
            <p>{ticket.description}</p>
          </div>
        ))}
      </section>
    </div>
  );
};

export default UserDashboard;
```

### Admin Analytics Component
```javascript
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const AdminAnalytics = () => {
  const [analytics, setAnalytics] = useState(null);
  const [riskUsers, setRiskUsers] = useState([]);
  const [anomalies, setAnomalies] = useState([]);

  useEffect(() => {
    fetchAnalytics();
  }, []);

  const fetchAnalytics = async () => {
    const token = localStorage.getItem('token');
    const config = {
      headers: { Authorization: `Bearer ${token}` }
    };

    try {
      // Fetch system analytics
      const analyticsRes = await axios.get(
        `${process.env.REACT_APP_API_URL}/admin/analytics`,
        config
      );
      setAnalytics(analyticsRes.data);

      // Fetch AI insights
      const riskRes = await axios.get(
        `${process.env.REACT_APP_ML_API_URL}/high-risk-users`,
        config
      );
      setRiskUsers(riskRes.data);

      const anomalyRes = await axios.get(
        `${process.env.REACT_APP_ML_API_URL}/recent-anomalies`,
        config
      );
      setAnomalies(anomalyRes.data);
    } catch (error) {
      console.error('Error fetching analytics:', error);
    }
  };

  if (!analytics) return <div>Loading analytics...</div>;

  return (
    <div className="admin-analytics">
      <h1>System Analytics</h1>

      <div className="metrics-grid">
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

      <section className="risk-section">
        <h2>High Risk Users</h2>
        <table>
          <thead>
            <tr>
              <th>User</th>
              <th>Risk Score</th>
              <th>Risk Level</th>
              <th>Action</th>
            </tr>
          </thead>
          <tbody>
            {riskUsers.map(user => (
              <tr key={user.id}>
                <td>{user.name}</td>
                <td>{user.riskScore.toFixed(2)}</td>
                <td>
                  <span className={`badge ${user.riskLevel}`}>
                    {user.riskLevel}
                  </span>
                </td>
                <td>
                  <button onClick={() => viewUserDetails(user.id)}>
                    View Details
                  </button>
                </td>
              </tr>
            ))}
          </tbody>
        </table>
      </section>

      <section className="anomaly-section">
        <h2>Recent Anomalies</h2>
        {anomalies.map(anomaly => (
          <div key={anomaly.id} className="anomaly-alert">
            <span className="timestamp">{anomaly.timestamp}</span>
            <h4>{anomaly.type}</h4>
            <p>{anomaly.description}</p>
            <p>User: {anomaly.userName} | IP: {anomaly.ipAddress}</p>
          </div>
        ))}
      </section>
    </div>
  );
};

const viewUserDetails = (userId) => {
  // Navigate to user detail page
  window.location.href = `/admin/users/${userId}`;
};

export default AdminAnalytics;
```

### Time Tracking Component
```javascript
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const TimeTracker = ({ taskId }) => {
  const [isRunning, setIsRunning] = useState(false);
  const [seconds, setSeconds] = useState(0);

  useEffect(() => {
    let interval = null;
    if (isRunning) {
      interval = setInterval(() => {
        setSeconds(s => s + 1);
      }, 1000);
    }
    return () => clearInterval(interval);
  }, [isRunning]);

  const startTimer = () => setIsRunning(true);
  
  const stopTimer = async () => {
    setIsRunning(false);
    await saveTimeLog();
  };

  const saveTimeLog = async () => {
    const token = localStorage.getItem('token');
    const minutes = Math.floor(seconds / 60);
    
    try {
      await axios.post(
        `${process.env.REACT_APP_API_URL}/tasks/${taskId}/time`,
        { 
          timeSpent: minutes,
          timestamp: new Date().toISOString()
        },
        { headers: { Authorization: `Bearer ${token}` } }
      );
      setSeconds(0);
    } catch (error) {
      console.error('Error saving time log:', error);
    }
  };

  const formatTime = (totalSeconds) => {
    const hours = Math.floor(totalSeconds / 3600);
    const minutes = Math.floor((totalSeconds % 3600) / 60);
    const secs = totalSeconds % 60;
    return `${hours.toString().padStart(2, '0')}:${minutes.toString().padStart(2, '0')}:${secs.toString().padStart(2, '0')}`;
  };

  return (
    <div className="time-tracker">
      <div className="timer-display">{formatTime(seconds)}</div>
      <div className="timer-controls">
        {!isRunning ? (
          <button onClick={startTimer} className="btn-start">Start</button>
        ) : (
          <button onClick={stopTimer} className="btn-stop">Stop & Save</button>
        )}
      </div>
    </div>
  );
};

export default TimeTracker;
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
      throw new Error('No authentication token provided');
    }

    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    const user = await User.findById(decoded.userId);

    if (!user) {
      throw new Error('User not found');
    }

    req.user = user;
    req.token = token;
    next();
  } catch (error) {
    res.status(401).json({ error: 'Please authenticate' });
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

// Get all users (Admin only)
exports.getAllUsers = async (req, res) => {
  try {
    const users = await User.find().select('-password');
    res.json(users);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
};

// Create user (Admin only)
exports.createUser = async (req, res) => {
  try {
    const { name, email, password, role } = req.body;
    
    const existingUser = await User.findOne({ email });
    if (existingUser) {
      return res.status(400).json({ error: 'Email already exists' });
    }

    const hashedPassword = await bcrypt.hash(password, 10);
    
    const user = new User({
      name,
      email,
      password: hashedPassword,
      role: role || 'user'
    });

    await user.save();
    
    res.status(201).json({
      message: 'User created successfully',
      user: { id: user._id, name: user.name, email: user.email, role: user.role }
    });
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
};

// Update user
exports.updateUser = async (req, res) => {
  try {
    const { id } = req.params;
    const updates = req.body;

    if (updates.password) {
      updates.password = await bcrypt.hash(updates.password, 10);
    }

    const user = await User.findByIdAndUpdate(
      id, 
      updates, 
      { new: true, runValidators: true }
    ).select('-password');

    if (!user) {
      return res.status(404).json({ error: 'User not found' });
    }

    res.json({ message: 'User updated successfully', user });
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
};

// Delete user
exports.deleteUser = async (req, res) => {
  try {
    const { id } = req.params;
    
    const user = await User.findByIdAndDelete(id);
    
    if (!user) {
      return res.status(404).json({ error: 'User not found' });
    }

    res.json({ message: 'User deleted successfully' });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
};
```

### Task Controller
```javascript
// controllers/taskController.js
const Task = require('../models/Task');

// Get user's tasks
exports.getMyTasks = async (req, res) => {
  try {
    const tasks = await Task.find({ assignedTo: req.user._id })
      .populate('assignedBy', 'name email')
      .sort({ createdAt: -1 });
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
};

// Create task
exports.createTask = async (req, res) => {
  try {
    const { title, description, assignedTo, priority, dueDate } = req.body;
    
    const task = new Task({
      title,
      description,
      assignedTo,
      assignedBy: req.user._id,
      priority: priority || 'medium',
      status: 'todo',
      dueDate
    });

    await task.save();
    
    res.status(201).json({
      message: 'Task created successfully',
      task
    });
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
};

// Update task status
exports.updateTaskStatus = async (req, res) => {
  try {
    const { id } = req.params;
    const { status } = req.body;

    const task = await Task.findById(id);
    
    if (!task) {
      return res.status(404).json({ error: 'Task not found' });
    }

    // Check if user has permission
    if (task.assignedTo.toString() !== req.user._id.toString() && 
        req.user.role !== 'admin') {
      return res.status(403).json({ error: 'Access denied' });
    }

    task.status = status;
    if (status === 'done') {
      task.completedAt = new Date();
    }
    
    await task.save();
    
    res.json({ message: 'Task status updated', task });
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
};

// Track time on task
exports.trackTime = async (req, res) => {
  try {
    const { id } = req.params;
    const { timeSpent, timestamp } = req.body;

    const task = await Task.findById(id);
    
    if (!task) {
      return res.status(404).json({ error: 'Task not found' });
    }

    task.timeLog.push({
      duration: timeSpent,
      timestamp: timestamp || new Date()
    });
    
    await task.save();
    
    res.json({ message: 'Time logged successfully', task });
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
};
```

## Python ML Service Implementation

### FastAPI Main Application
```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException, Depends
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from typing import List, Optional
import numpy as np
from sklearn.ensemble import RandomForestClassifier, IsolationForest
import joblib
import os

app = FastAPI(title="Enterprise User Management AI Service")

# CORS middleware
app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:3000"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Models
risk_model = None
anomaly_detector = None
burnout_model = None

@app.on_event("startup")
async def load_models():
    global risk_model, anomaly_detector, burnout_model
    model_path = os.getenv("MODEL_PATH", "./models")
    
    try:
        risk_model = joblib.load(f"{model_path}/risk_model.pkl")
        anomaly_detector = joblib.load(f"{model_path}/anomaly_detector.pkl")
        burnout_model = joblib.load(f"{model_path}/burnout_model.pkl")
    except FileNotFoundError:
        # Initialize new models if not found
        risk_model = RandomForestClassifier(n_estimators=100)
        anomaly_detector = IsolationForest(contamination=0.1)
        burnout_model = RandomForestClassifier(n_estimators=100)

# Request models
class RiskPredictionRequest(BaseModel):
    user_id: str
    task_completion_rate: float
    average_task_time: float
    overdue_tasks: int
    login_frequency: float

class AnomalyDetectionRequest(BaseModel):
    user_id: str
    login_time: str
    ip_
