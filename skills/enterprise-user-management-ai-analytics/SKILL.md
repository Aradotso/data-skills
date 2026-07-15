---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for risk detection, burnout analysis, and ticket classification
triggers:
  - "set up enterprise user management system"
  - "implement AI-powered user analytics"
  - "create user management dashboard with ML"
  - "integrate AI ticket classification system"
  - "build admin dashboard with anomaly detection"
  - "add burnout prediction to user system"
  - "configure ML service for user management"
  - "deploy enterprise user management app"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

The Enterprise User Management System is a full-stack application that combines traditional user/task management with AI-powered analytics. It provides:

- **User Management**: Role-based access control, authentication via JWT
- **Task Management**: Kanban boards, time tracking, assignment workflows
- **Support System**: Ticket creation, tracking, and AI-based classification
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, project delay prediction
- **Admin Tools**: User oversight, audit logs, organization analytics

The system consists of three main components:
- **Frontend**: React.js application (port 3000)
- **Backend**: Node.js REST API (port 5000)
- **ML Service**: FastAPI-based Python service (port 8000)

## Installation

### Prerequisites

- Node.js 14+ and npm
- Python 3.8+
- MongoDB running locally or connection string available

### Clone and Setup

```bash
# Clone repository
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics

# Setup backend
cd backend
npm install
cp .env.example .env  # Configure environment variables

# Setup ML service
cd ../ml-service
pip install -r requirements.txt

# Setup frontend
cd ../frontend
npm install
```

### Environment Configuration

**Backend (.env)**:
```env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
JWT_SECRET=your_jwt_secret_key_here
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
```

**ML Service (.env)**:
```env
MODEL_PATH=./models
DB_CONNECTION=mongodb://localhost:27017/enterprise_user_mgmt
LOG_LEVEL=INFO
```

**Frontend (.env)**:
```env
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_URL=http://localhost:8000
```

## Running the System

Start each service in separate terminals:

```bash
# Terminal 1: Backend
cd backend
npm start

# Terminal 2: ML Service
cd ml-service
uvicorn main:app --reload

# Terminal 3: Frontend
cd frontend
npm start
```

Access the application at `http://localhost:3000`

## Backend API Usage

### Authentication

```javascript
// backend/controllers/authController.js
const jwt = require('jsonwebtoken');
const bcrypt = require('bcryptjs');
const User = require('../models/User');

// Register new user
const register = async (req, res) => {
  try {
    const { username, email, password, role } = req.body;
    
    // Hash password
    const salt = await bcrypt.genSalt(10);
    const hashedPassword = await bcrypt.hash(password, salt);
    
    const user = new User({
      username,
      email,
      password: hashedPassword,
      role: role || 'user'
    });
    
    await user.save();
    
    // Generate token
    const token = jwt.sign(
      { id: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: '24h' }
    );
    
    res.status(201).json({
      success: true,
      token,
      user: { id: user._id, username, email, role: user.role }
    });
  } catch (error) {
    res.status(500).json({ success: false, message: error.message });
  }
};

// Login
const login = async (req, res) => {
  try {
    const { email, password } = req.body;
    
    const user = await User.findOne({ email });
    if (!user) {
      return res.status(401).json({ success: false, message: 'Invalid credentials' });
    }
    
    const isMatch = await bcrypt.compare(password, user.password);
    if (!isMatch) {
      return res.status(401).json({ success: false, message: 'Invalid credentials' });
    }
    
    const token = jwt.sign(
      { id: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: '24h' }
    );
    
    res.json({
      success: true,
      token,
      user: { id: user._id, username: user.username, email, role: user.role }
    });
  } catch (error) {
    res.status(500).json({ success: false, message: error.message });
  }
};

module.exports = { register, login };
```

### User Management

```javascript
// backend/controllers/userController.js
const User = require('../models/User');

// Get all users (Admin only)
const getAllUsers = async (req, res) => {
  try {
    const users = await User.find().select('-password');
    res.json({ success: true, users });
  } catch (error) {
    res.status(500).json({ success: false, message: error.message });
  }
};

// Update user
const updateUser = async (req, res) => {
  try {
    const { id } = req.params;
    const updates = req.body;
    
    // Prevent password update through this endpoint
    delete updates.password;
    
    const user = await User.findByIdAndUpdate(
      id,
      { $set: updates },
      { new: true, runValidators: true }
    ).select('-password');
    
    if (!user) {
      return res.status(404).json({ success: false, message: 'User not found' });
    }
    
    res.json({ success: true, user });
  } catch (error) {
    res.status(500).json({ success: false, message: error.message });
  }
};

// Delete user
const deleteUser = async (req, res) => {
  try {
    const { id } = req.params;
    const user = await User.findByIdAndDelete(id);
    
    if (!user) {
      return res.status(404).json({ success: false, message: 'User not found' });
    }
    
    res.json({ success: true, message: 'User deleted successfully' });
  } catch (error) {
    res.status(500).json({ success: false, message: error.message });
  }
};

module.exports = { getAllUsers, updateUser, deleteUser };
```

### Task Management

```javascript
// backend/controllers/taskController.js
const Task = require('../models/Task');

// Create task
const createTask = async (req, res) => {
  try {
    const { title, description, assignedTo, priority, dueDate } = req.body;
    
    const task = new Task({
      title,
      description,
      assignedTo,
      priority: priority || 'medium',
      status: 'todo',
      dueDate,
      createdBy: req.user.id
    });
    
    await task.save();
    await task.populate('assignedTo', 'username email');
    
    res.status(201).json({ success: true, task });
  } catch (error) {
    res.status(500).json({ success: false, message: error.message });
  }
};

// Get user tasks
const getUserTasks = async (req, res) => {
  try {
    const tasks = await Task.find({ assignedTo: req.user.id })
      .populate('createdBy', 'username')
      .sort({ createdAt: -1 });
    
    res.json({ success: true, tasks });
  } catch (error) {
    res.status(500).json({ success: false, message: error.message });
  }
};

// Update task status
const updateTaskStatus = async (req, res) => {
  try {
    const { id } = req.params;
    const { status } = req.body;
    
    const task = await Task.findByIdAndUpdate(
      id,
      { status, updatedAt: Date.now() },
      { new: true }
    ).populate('assignedTo', 'username email');
    
    if (!task) {
      return res.status(404).json({ success: false, message: 'Task not found' });
    }
    
    res.json({ success: true, task });
  } catch (error) {
    res.status(500).json({ success: false, message: error.message });
  }
};

module.exports = { createTask, getUserTasks, updateTaskStatus };
```

### Support Tickets

```javascript
// backend/controllers/ticketController.js
const Ticket = require('../models/Ticket');
const axios = require('axios');

// Create ticket with AI classification
const createTicket = async (req, res) => {
  try {
    const { title, description, priority } = req.body;
    
    // Call ML service for classification
    let category = 'general';
    let aiPriority = priority;
    
    try {
      const mlResponse = await axios.post(
        `${process.env.ML_SERVICE_URL}/classify-ticket`,
        { title, description }
      );
      category = mlResponse.data.category;
      aiPriority = mlResponse.data.suggested_priority;
    } catch (mlError) {
      console.error('ML classification failed:', mlError.message);
    }
    
    const ticket = new Ticket({
      title,
      description,
      category,
      priority: aiPriority || priority,
      status: 'open',
      createdBy: req.user.id
    });
    
    await ticket.save();
    await ticket.populate('createdBy', 'username email');
    
    res.status(201).json({ success: true, ticket });
  } catch (error) {
    res.status(500).json({ success: false, message: error.message });
  }
};

// Get all tickets (Admin)
const getAllTickets = async (req, res) => {
  try {
    const { status, category } = req.query;
    const filter = {};
    
    if (status) filter.status = status;
    if (category) filter.category = category;
    
    const tickets = await Ticket.find(filter)
      .populate('createdBy', 'username email')
      .populate('assignedTo', 'username email')
      .sort({ createdAt: -1 });
    
    res.json({ success: true, tickets });
  } catch (error) {
    res.status(500).json({ success: false, message: error.message });
  }
};

module.exports = { createTicket, getAllTickets };
```

## ML Service API Usage

### FastAPI Setup

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from typing import Optional, List
import joblib
import numpy as np
from datetime import datetime
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

# Load models
MODEL_PATH = os.getenv("MODEL_PATH", "./models")

class TicketClassificationRequest(BaseModel):
    title: str
    description: str

class RiskPredictionRequest(BaseModel):
    user_id: str
    login_frequency: float
    failed_logins: int
    data_access_count: int
    after_hours_activity: int
    location_changes: int

class BurnoutAnalysisRequest(BaseModel):
    user_id: str
    tasks_completed: int
    avg_task_duration: float
    overtime_hours: float
    tasks_overdue: int
    days_since_break: int

@app.get("/")
async def root():
    return {"message": "Enterprise User Management ML Service", "status": "running"}

@app.post("/classify-ticket")
async def classify_ticket(request: TicketClassificationRequest):
    """Classify support ticket and suggest priority"""
    try:
        # Simple rule-based classification (replace with ML model)
        text = (request.title + " " + request.description).lower()
        
        # Category classification
        if any(word in text for word in ["password", "login", "access", "account"]):
            category = "authentication"
            priority = "high"
        elif any(word in text for word in ["bug", "error", "crash", "broken"]):
            category = "technical"
            priority = "high"
        elif any(word in text for word in ["feature", "request", "suggestion"]):
            category = "feature_request"
            priority = "low"
        elif any(word in text for word in ["billing", "payment", "invoice"]):
            category = "billing"
            priority = "medium"
        else:
            category = "general"
            priority = "medium"
        
        return {
            "category": category,
            "suggested_priority": priority,
            "confidence": 0.85
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/predict-risk")
async def predict_risk(request: RiskPredictionRequest):
    """Predict user risk score based on behavior"""
    try:
        # Calculate risk score (0-100)
        risk_score = 0
        risk_factors = []
        
        # Failed logins contribute to risk
        if request.failed_logins > 5:
            risk_score += 30
            risk_factors.append("Multiple failed login attempts")
        
        # Excessive after-hours activity
        if request.after_hours_activity > 20:
            risk_score += 25
            risk_factors.append("High after-hours activity")
        
        # Frequent location changes
        if request.location_changes > 10:
            risk_score += 20
            risk_factors.append("Frequent location changes")
        
        # Abnormal data access
        if request.data_access_count > 100:
            risk_score += 25
            risk_factors.append("Unusually high data access")
        
        risk_level = "low"
        if risk_score >= 70:
            risk_level = "high"
        elif risk_score >= 40:
            risk_level = "medium"
        
        return {
            "user_id": request.user_id,
            "risk_score": min(risk_score, 100),
            "risk_level": risk_level,
            "risk_factors": risk_factors,
            "timestamp": datetime.now().isoformat()
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/analyze-burnout")
async def analyze_burnout(request: BurnoutAnalysisRequest):
    """Analyze user burnout risk"""
    try:
        burnout_score = 0
        indicators = []
        
        # Excessive overtime
        if request.overtime_hours > 15:
            burnout_score += 30
            indicators.append("High overtime hours")
        
        # High task duration
        if request.avg_task_duration > 8:
            burnout_score += 20
            indicators.append("Extended task completion times")
        
        # Overdue tasks
        if request.tasks_overdue > 5:
            burnout_score += 25
            indicators.append("Multiple overdue tasks")
        
        # No breaks
        if request.days_since_break > 30:
            burnout_score += 25
            indicators.append("No recent breaks")
        
        burnout_level = "low"
        if burnout_score >= 70:
            burnout_level = "high"
        elif burnout_score >= 40:
            burnout_level = "medium"
        
        recommendations = []
        if burnout_level in ["medium", "high"]:
            recommendations.extend([
                "Schedule time off",
                "Redistribute workload",
                "Review task priorities"
            ])
        
        return {
            "user_id": request.user_id,
            "burnout_score": min(burnout_score, 100),
            "burnout_level": burnout_level,
            "indicators": indicators,
            "recommendations": recommendations,
            "timestamp": datetime.now().isoformat()
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/predict-project-delay")
async def predict_project_delay(
    tasks_total: int,
    tasks_completed: int,
    days_remaining: int,
    avg_completion_rate: float
):
    """Predict if project will be delayed"""
    try:
        tasks_remaining = tasks_total - tasks_completed
        required_rate = tasks_remaining / days_remaining if days_remaining > 0 else float('inf')
        
        delay_probability = 0
        if required_rate > avg_completion_rate * 1.5:
            delay_probability = 0.9
        elif required_rate > avg_completion_rate:
            delay_probability = 0.6
        else:
            delay_probability = 0.2
        
        estimated_delay_days = max(0, int((tasks_remaining / avg_completion_rate) - days_remaining))
        
        return {
            "delay_probability": delay_probability,
            "estimated_delay_days": estimated_delay_days,
            "required_completion_rate": round(required_rate, 2),
            "current_completion_rate": round(avg_completion_rate, 2),
            "recommendation": "Increase resources" if delay_probability > 0.5 else "On track"
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

## Frontend Integration

### API Service Setup

```javascript
// frontend/src/services/api.js
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL || 'http://localhost:5000/api';
const ML_URL = process.env.REACT_APP_ML_URL || 'http://localhost:8000';

// Create axios instance with auth
const api = axios.create({
  baseURL: API_URL,
});

// Add auth token to requests
api.interceptors.request.use((config) => {
  const token = localStorage.getItem('token');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

// Auth services
export const authService = {
  login: async (email, password) => {
    const response = await api.post('/auth/login', { email, password });
    if (response.data.token) {
      localStorage.setItem('token', response.data.token);
      localStorage.setItem('user', JSON.stringify(response.data.user));
    }
    return response.data;
  },
  
  register: async (userData) => {
    const response = await api.post('/auth/register', userData);
    return response.data;
  },
  
  logout: () => {
    localStorage.removeItem('token');
    localStorage.removeItem('user');
  }
};

// User services
export const userService = {
  getAllUsers: async () => {
    const response = await api.get('/users');
    return response.data.users;
  },
  
  updateUser: async (userId, updates) => {
    const response = await api.put(`/users/${userId}`, updates);
    return response.data.user;
  },
  
  deleteUser: async (userId) => {
    const response = await api.delete(`/users/${userId}`);
    return response.data;
  }
};

// Task services
export const taskService = {
  getUserTasks: async () => {
    const response = await api.get('/tasks/my-tasks');
    return response.data.tasks;
  },
  
  createTask: async (taskData) => {
    const response = await api.post('/tasks', taskData);
    return response.data.task;
  },
  
  updateTaskStatus: async (taskId, status) => {
    const response = await api.patch(`/tasks/${taskId}/status`, { status });
    return response.data.task;
  }
};

// Ticket services
export const ticketService = {
  createTicket: async (ticketData) => {
    const response = await api.post('/tickets', ticketData);
    return response.data.ticket;
  },
  
  getAllTickets: async (filters = {}) => {
    const response = await api.get('/tickets', { params: filters });
    return response.data.tickets;
  }
};

// ML services
export const mlService = {
  classifyTicket: async (title, description) => {
    const response = await axios.post(`${ML_URL}/classify-ticket`, {
      title,
      description
    });
    return response.data;
  },
  
  predictRisk: async (userData) => {
    const response = await axios.post(`${ML_URL}/predict-risk`, userData);
    return response.data;
  },
  
  analyzeBurnout: async (userData) => {
    const response = await axios.post(`${ML_URL}/analyze-burnout`, userData);
    return response.data;
  },
  
  predictProjectDelay: async (projectData) => {
    const response = await axios.post(`${ML_URL}/predict-project-delay`, null, {
      params: projectData
    });
    return response.data;
  }
};

export default api;
```

### Admin Dashboard Component

```javascript
// frontend/src/components/AdminDashboard.jsx
import React, { useState, useEffect } from 'react';
import { userService, mlService } from '../services/api';
import './AdminDashboard.css';

const AdminDashboard = () => {
  const [users, setUsers] = useState([]);
  const [riskAnalysis, setRiskAnalysis] = useState([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    loadDashboardData();
  }, []);

  const loadDashboardData = async () => {
    try {
      setLoading(true);
      const usersData = await userService.getAllUsers();
      setUsers(usersData);
      
      // Analyze risk for each user
      const riskPromises = usersData.map(async (user) => {
        try {
          const risk = await mlService.predictRisk({
            user_id: user._id,
            login_frequency: user.loginFrequency || 5,
            failed_logins: user.failedLogins || 0,
            data_access_count: user.dataAccessCount || 50,
            after_hours_activity: user.afterHoursActivity || 5,
            location_changes: user.locationChanges || 2
          });
          return { ...user, risk };
        } catch (err) {
          return { ...user, risk: null };
        }
      });
      
      const analysisResults = await Promise.all(riskPromises);
      setRiskAnalysis(analysisResults);
    } catch (error) {
      console.error('Failed to load dashboard:', error);
    } finally {
      setLoading(false);
    }
  };

  const handleDeleteUser = async (userId) => {
    if (window.confirm('Are you sure you want to delete this user?')) {
      try {
        await userService.deleteUser(userId);
        loadDashboardData();
      } catch (error) {
        alert('Failed to delete user');
      }
    }
  };

  if (loading) return <div className="loading">Loading dashboard...</div>;

  return (
    <div className="admin-dashboard">
      <h1>Admin Dashboard</h1>
      
      <div className="stats-grid">
        <div className="stat-card">
          <h3>Total Users</h3>
          <p className="stat-number">{users.length}</p>
        </div>
        <div className="stat-card">
          <h3>High Risk Users</h3>
          <p className="stat-number alert">
            {riskAnalysis.filter(u => u.risk?.risk_level === 'high').length}
          </p>
        </div>
        <div className="stat-card">
          <h3>Active Users</h3>
          <p className="stat-number">
            {users.filter(u => u.status === 'active').length}
          </p>
        </div>
      </div>

      <div className="users-section">
        <h2>User Risk Analysis</h2>
        <table className="users-table">
          <thead>
            <tr>
              <th>Username</th>
              <th>Email</th>
              <th>Role</th>
              <th>Risk Level</th>
              <th>Risk Score</th>
              <th>Actions</th>
            </tr>
          </thead>
          <tbody>
            {riskAnalysis.map((user) => (
              <tr key={user._id}>
                <td>{user.username}</td>
                <td>{user.email}</td>
                <td>{user.role}</td>
                <td>
                  <span className={`risk-badge ${user.risk?.risk_level || 'unknown'}`}>
                    {user.risk?.risk_level || 'N/A'}
                  </span>
                </td>
                <td>{user.risk?.risk_score || 'N/A'}</td>
                <td>
                  <button 
                    className="btn-danger"
                    onClick={() => handleDeleteUser(user._id)}
                  >
                    Delete
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

### User Dashboard with Kanban Board

```javascript
// frontend/src/components/UserDashboard.jsx
import React, { useState, useEffect } from 'react';
import { taskService, mlService } from '../services/api';
import './UserDashboard.css';

const UserDashboard = () => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });
  const [burnoutAnalysis, setBurnoutAnalysis] = useState(null);

  useEffect(() => {
    loadUserData();
  }, []);

  const loadUserData = async () => {
    try {
      const tasksData = await taskService.getUserTasks();
      
      // Organize tasks by status
      const organized = {
        todo: tasksData.filter(t => t.status === 'todo'),
        inProgress: tasksData.filter(t => t.status === 'in_progress'),
        done: tasksData.filter(t => t.status === 'done')
      };
      setTasks(organized);
      
      // Analyze burnout
      const user = JSON.parse(localStorage.getItem('user'));
      const burnout = await mlService.analyzeBurnout({
        user_id: user.id,
        tasks_completed: organized.done.length,
        avg_task_duration: 6.5,
        overtime_hours: 12,
        tasks_overdue: tasksData.filter(t => new Date(t.dueDate) < new Date()).length,
        days_since_break: 25
      });
      setBurnoutAnalysis(burnout);
    } catch (error) {
      console.error('Failed to load user data:', error);
    }
  };

  const handleStatusChange = async (taskId, newStatus) => {
    try {
      await taskService.updateTaskStatus(taskId, newStatus);
      loadUserData();
    } catch (error) {
      alert('Failed to update task status');
    }
  };

  const TaskCard = ({ task, onStatusChange }) => (
    <div className="task-card">
      <h4>{task.title}</h4>
      <p>{task.description}</p>
      <div className="task-meta">
        <span className={`priority ${task.priority}`}>{task.priority}</span>
        <span className="due-date">
          Due: {new Date(task.dueDate).toLocaleDateString()}
        </span>
      </div>
      <select 
        value={task.status}
        onChange={(e) => onStatusChange(task._id, e.target.value)}
        className="status-select"
      >
        <option value="todo">To Do</option>
        <option value="in_progress">In Progress</option>
        <option value="done">Done</option>
      </select>
    </div>
  );

  return (
    <div className="user-dashboard">
      <h1>My Dashboard</h1>
      
      {burnoutAnalysis && (
        <div className={`burnout-alert ${burnoutAnalysis.burnout_level}`}>
          <h3>Burnout Analysis</h3>
          <p>Level: {burnoutAnalysis.burnout_level.toUpperCase()}</p>
          <p>Score: {burnoutAnalysis.burnout_score}/100</p>
          {burnoutAnalysis.recommendations.length > 0 && (
            <ul>
              {burnoutAnalysis.recommendations.map((rec, idx) => (
                <li key={idx}>{rec}</li>
              ))}
            </ul>
          )}
        </div>
      )}

      <div className="kanban-board">
        <div className="kanban-column">
          <h2>To Do ({tasks.todo.length})</h2>
          {tasks.todo.map(task => (
            <TaskCard 
              key={task._id} 
              task={task} 
              onStatusChange={handleStatusChange}
            />
          ))}
        </div>

        <div className="kanban-column">
          <h2>In Progress ({tasks.inProgress.length})</h2>
          {tasks.inProgress.map(task => (
            <TaskCard 
              key={task._id} 
              task={task} 
              onStatusChange={handleStatusChange}
            />
          ))}
        </div>

        <div className="kanban-column">
          <h2>Done ({tasks.done.length})</h2>
          {tasks.done.map(task => (
            <TaskCard 
              key={task._id} 
              task={task} 
              onStatusChange={handleStatusChange}
            />
          ))}
        </div>
      </div>
    </div>
  );
};

export default UserDashboard;
```

## Common Patterns

### Middleware for Authentication

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

const auth = async (req, res
