---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics, risk detection, and task management capabilities
triggers:
  - "set up enterprise user management system"
  - "implement AI analytics for user management"
  - "create user management dashboard with ML"
  - "integrate AI-powered ticket classification"
  - "build task management with burnout detection"
  - "deploy user management system with FastAPI ML service"
  - "configure JWT authentication for enterprise app"
  - "implement anomaly detection in user management"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System with AI Analytics is a full-stack JavaScript/Node.js application that combines traditional user management with machine learning capabilities. It provides role-based access control, task management with Kanban boards, support ticket systems, and AI-powered features including risk detection, anomaly detection, burnout analysis, and predictive project insights.

**Key Components:**
- **Frontend**: React.js with modern UI for user/admin dashboards
- **Backend**: Node.js with Express, REST APIs, JWT authentication
- **ML Service**: FastAPI + scikit-learn + River for real-time ML predictions
- **Database**: MongoDB for user, task, and ticket data

## Installation

### Prerequisites

```bash
# Node.js 14+ and npm
node --version
npm --version

# Python 3.8+ for ML service
python --version

# MongoDB running locally or cloud instance
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

## Configuration

### Backend Configuration

Create `backend/.env`:

```bash
# Server Configuration
PORT=5000
NODE_ENV=development

# MongoDB Configuration
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
# OR for MongoDB Atlas
# MONGODB_URI=mongodb+srv://<username>:<password>@cluster.mongodb.net/enterprise-user-mgmt

# JWT Configuration
JWT_SECRET=your_jwt_secret_key_here
JWT_EXPIRE=7d

# ML Service URL
ML_SERVICE_URL=http://localhost:8000

# CORS Configuration
FRONTEND_URL=http://localhost:3000
```

### Frontend Configuration

Create `frontend/.env`:

```bash
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_API_URL=http://localhost:8000
```

### ML Service Configuration

Create `ml-service/.env`:

```bash
# FastAPI Configuration
API_HOST=0.0.0.0
API_PORT=8000

# Model Configuration
MODEL_PATH=./models
ENABLE_ONLINE_LEARNING=true

# Backend API URL
BACKEND_API_URL=http://localhost:5000/api
```

## Running the Application

### Start All Services

```bash
# Terminal 1 - Backend
cd backend
npm start
# Runs at http://localhost:5000

# Terminal 2 - Frontend
cd frontend
npm start
# Runs at http://localhost:3000

# Terminal 3 - ML Service
cd ml-service
uvicorn main:app --reload --host 0.0.0.0 --port 8000
# Runs at http://localhost:8000
```

### Development Mode

```bash
# Backend with nodemon
cd backend
npm run dev

# Frontend with hot reload (default)
cd frontend
npm start

# ML Service with auto-reload (already included with --reload flag)
cd ml-service
uvicorn main:app --reload
```

## Backend API Reference

### Authentication Endpoints

```javascript
// Register new user
POST /api/auth/register
Content-Type: application/json

{
  "username": "johndoe",
  "email": "john@example.com",
  "password": "securePassword123",
  "role": "user" // or "admin"
}

// Login
POST /api/auth/login
{
  "email": "john@example.com",
  "password": "securePassword123"
}
// Returns: { token: "jwt_token", user: {...} }

// Get current user
GET /api/auth/me
Authorization: Bearer <token>
```

### User Management (Admin)

```javascript
// Get all users
GET /api/users
Authorization: Bearer <admin_token>

// Get user by ID
GET /api/users/:id
Authorization: Bearer <admin_token>

// Update user
PUT /api/users/:id
Authorization: Bearer <admin_token>
{
  "username": "newusername",
  "email": "newemail@example.com",
  "role": "admin"
}

// Delete user
DELETE /api/users/:id
Authorization: Bearer <admin_token>
```

### Task Management

```javascript
// Create task
POST /api/tasks
Authorization: Bearer <token>
{
  "title": "Implement user dashboard",
  "description": "Build React dashboard component",
  "assignedTo": "user_id",
  "priority": "high",
  "dueDate": "2026-05-01",
  "status": "todo" // todo, in-progress, done
}

// Get user tasks
GET /api/tasks/user/:userId
Authorization: Bearer <token>

// Update task status
PUT /api/tasks/:id
{
  "status": "in-progress",
  "timeSpent": 120 // minutes
}

// Track time on task
POST /api/tasks/:id/track-time
{
  "minutes": 30
}
```

### Support Ticket System

```javascript
// Create ticket
POST /api/tickets
Authorization: Bearer <token>
{
  "title": "Login issue",
  "description": "Cannot login with correct credentials",
  "priority": "high",
  "category": "technical"
}

// Get all tickets (admin)
GET /api/tickets
Authorization: Bearer <admin_token>

// Update ticket (admin)
PUT /api/tickets/:id
Authorization: Bearer <admin_token>
{
  "status": "resolved",
  "assignedTo": "admin_user_id",
  "aiClassification": "authentication_issue"
}
```

## ML Service API Reference

### AI Analytics Endpoints

```python
# Risk Prediction
POST http://localhost:8000/predict/risk
Content-Type: application/json

{
  "userId": "user_id_123",
  "features": {
    "loginFrequency": 5,
    "taskCompletionRate": 0.85,
    "avgTaskTime": 120,
    "failedLoginAttempts": 0,
    "dataAccessPatterns": [1, 2, 3, 1, 2]
  }
}
# Returns: { "riskScore": 0.23, "riskLevel": "low" }

# Anomaly Detection
POST http://localhost:8000/detect/anomaly
{
  "userId": "user_id_123",
  "activity": {
    "timestamp": "2026-04-15T10:30:00Z",
    "action": "data_export",
    "ipAddress": "192.168.1.1",
    "deviceInfo": "Chrome/Linux"
  }
}
# Returns: { "isAnomaly": false, "anomalyScore": 0.12 }

# Burnout Detection
POST http://localhost:8000/analyze/burnout
{
  "userId": "user_id_123",
  "workload": {
    "tasksAssigned": 15,
    "tasksCompleted": 12,
    "avgWorkHoursPerDay": 9.5,
    "weekendWork": true,
    "overtimeHours": 10
  }
}
# Returns: { "burnoutRisk": "medium", "score": 0.62, "recommendations": [...] }

# Ticket Classification
POST http://localhost:8000/classify/ticket
{
  "ticketId": "ticket_123",
  "title": "Cannot access dashboard",
  "description": "Getting 403 error when trying to access admin dashboard",
  "priority": "high"
}
# Returns: { "category": "access_control", "suggestedAssignee": "admin_id", "confidence": 0.89 }

# Project Delay Prediction
POST http://localhost:8000/predict/project-delay
{
  "projectId": "proj_123",
  "metrics": {
    "tasksTotal": 50,
    "tasksCompleted": 30,
    "daysRemaining": 10,
    "teamSize": 5,
    "avgVelocity": 3.2
  }
}
# Returns: { "delayProbability": 0.68, "estimatedDelay": 5, "delayUnit": "days" }
```

## Frontend Usage Patterns

### React Component Examples

#### Admin Dashboard Component

```javascript
// src/components/AdminDashboard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const AdminDashboard = () => {
  const [users, setUsers] = useState([]);
  const [analytics, setAnalytics] = useState(null);
  const token = localStorage.getItem('token');

  useEffect(() => {
    fetchUsers();
    fetchAnalytics();
  }, []);

  const fetchUsers = async () => {
    try {
      const response = await axios.get(
        `${process.env.REACT_APP_API_URL}/users`,
        { headers: { Authorization: `Bearer ${token}` } }
      );
      setUsers(response.data);
    } catch (error) {
      console.error('Error fetching users:', error);
    }
  };

  const fetchAnalytics = async () => {
    try {
      const response = await axios.get(
        `${process.env.REACT_APP_API_URL}/analytics/overview`,
        { headers: { Authorization: `Bearer ${token}` } }
      );
      setAnalytics(response.data);
    } catch (error) {
      console.error('Error fetching analytics:', error);
    }
  };

  const deleteUser = async (userId) => {
    if (!window.confirm('Are you sure?')) return;
    
    try {
      await axios.delete(
        `${process.env.REACT_APP_API_URL}/users/${userId}`,
        { headers: { Authorization: `Bearer ${token}` } }
      );
      fetchUsers();
    } catch (error) {
      console.error('Error deleting user:', error);
    }
  };

  return (
    <div className="admin-dashboard">
      <h1>Admin Dashboard</h1>
      
      {analytics && (
        <div className="analytics-overview">
          <div className="stat-card">
            <h3>Total Users</h3>
            <p>{analytics.totalUsers}</p>
          </div>
          <div className="stat-card">
            <h3>Active Tasks</h3>
            <p>{analytics.activeTasks}</p>
          </div>
          <div className="stat-card">
            <h3>Open Tickets</h3>
            <p>{analytics.openTickets}</p>
          </div>
        </div>
      )}

      <div className="user-list">
        <h2>Users</h2>
        <table>
          <thead>
            <tr>
              <th>Username</th>
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
                  <button onClick={() => deleteUser(user._id)}>Delete</button>
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

#### Kanban Board Component

```javascript
// src/components/KanbanBoard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import { DragDropContext, Droppable, Draggable } from 'react-beautiful-dnd';

const KanbanBoard = ({ userId }) => {
  const [tasks, setTasks] = useState({
    todo: [],
    'in-progress': [],
    done: []
  });
  const token = localStorage.getItem('token');

  useEffect(() => {
    fetchTasks();
  }, [userId]);

  const fetchTasks = async () => {
    try {
      const response = await axios.get(
        `${process.env.REACT_APP_API_URL}/tasks/user/${userId}`,
        { headers: { Authorization: `Bearer ${token}` } }
      );
      
      const categorized = {
        todo: response.data.filter(t => t.status === 'todo'),
        'in-progress': response.data.filter(t => t.status === 'in-progress'),
        done: response.data.filter(t => t.status === 'done')
      };
      
      setTasks(categorized);
    } catch (error) {
      console.error('Error fetching tasks:', error);
    }
  };

  const onDragEnd = async (result) => {
    if (!result.destination) return;

    const { source, destination, draggableId } = result;
    
    if (source.droppableId === destination.droppableId) return;

    // Update UI optimistically
    const sourceTasks = [...tasks[source.droppableId]];
    const destTasks = [...tasks[destination.droppableId]];
    const [movedTask] = sourceTasks.splice(source.index, 1);
    destTasks.splice(destination.index, 0, movedTask);

    setTasks({
      ...tasks,
      [source.droppableId]: sourceTasks,
      [destination.droppableId]: destTasks
    });

    // Update backend
    try {
      await axios.put(
        `${process.env.REACT_APP_API_URL}/tasks/${draggableId}`,
        { status: destination.droppableId },
        { headers: { Authorization: `Bearer ${token}` } }
      );
    } catch (error) {
      console.error('Error updating task:', error);
      fetchTasks(); // Revert on error
    }
  };

  return (
    <DragDropContext onDragEnd={onDragEnd}>
      <div className="kanban-board">
        {['todo', 'in-progress', 'done'].map(columnId => (
          <Droppable key={columnId} droppableId={columnId}>
            {(provided) => (
              <div
                className="kanban-column"
                ref={provided.innerRef}
                {...provided.droppableProps}
              >
                <h3>{columnId.toUpperCase().replace('-', ' ')}</h3>
                {tasks[columnId].map((task, index) => (
                  <Draggable key={task._id} draggableId={task._id} index={index}>
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

#### AI Analytics Integration

```javascript
// src/components/AIInsights.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const AIInsights = ({ userId }) => {
  const [insights, setInsights] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchAIInsights();
  }, [userId]);

  const fetchAIInsights = async () => {
    setLoading(true);
    try {
      // Fetch user metrics from backend
      const metricsResponse = await axios.get(
        `${process.env.REACT_APP_API_URL}/users/${userId}/metrics`,
        { headers: { Authorization: `Bearer ${localStorage.getItem('token')}` } }
      );

      // Get AI predictions
      const [riskPrediction, burnoutAnalysis, anomalyCheck] = await Promise.all([
        axios.post(`${process.env.REACT_APP_ML_API_URL}/predict/risk`, {
          userId,
          features: metricsResponse.data
        }),
        axios.post(`${process.env.REACT_APP_ML_API_URL}/analyze/burnout`, {
          userId,
          workload: metricsResponse.data.workload
        }),
        axios.post(`${process.env.REACT_APP_ML_API_URL}/detect/anomaly`, {
          userId,
          activity: metricsResponse.data.recentActivity
        })
      ]);

      setInsights({
        risk: riskPrediction.data,
        burnout: burnoutAnalysis.data,
        anomaly: anomalyCheck.data
      });
    } catch (error) {
      console.error('Error fetching AI insights:', error);
    } finally {
      setLoading(false);
    }
  };

  if (loading) return <div>Loading AI insights...</div>;
  if (!insights) return <div>No insights available</div>;

  return (
    <div className="ai-insights">
      <h2>AI-Powered Insights</h2>
      
      <div className="insight-card risk">
        <h3>Risk Assessment</h3>
        <div className={`risk-level ${insights.risk.riskLevel}`}>
          {insights.risk.riskLevel.toUpperCase()}
        </div>
        <p>Risk Score: {(insights.risk.riskScore * 100).toFixed(1)}%</p>
      </div>

      <div className="insight-card burnout">
        <h3>Burnout Analysis</h3>
        <div className={`burnout-level ${insights.burnout.burnoutRisk}`}>
          {insights.burnout.burnoutRisk.toUpperCase()} RISK
        </div>
        <p>Score: {(insights.burnout.score * 100).toFixed(1)}%</p>
        {insights.burnout.recommendations && (
          <ul>
            {insights.burnout.recommendations.map((rec, i) => (
              <li key={i}>{rec}</li>
            ))}
          </ul>
        )}
      </div>

      <div className="insight-card anomaly">
        <h3>Activity Monitoring</h3>
        <p>
          {insights.anomaly.isAnomaly 
            ? '⚠️ Unusual activity detected' 
            : '✓ Activity normal'}
        </p>
        <p>Anomaly Score: {(insights.anomaly.anomalyScore * 100).toFixed(1)}%</p>
      </div>
    </div>
  );
};

export default AIInsights;
```

## Backend Implementation Patterns

### User Controller

```javascript
// backend/controllers/userController.js
const User = require('../models/User');
const jwt = require('jsonwebtoken');
const bcrypt = require('bcryptjs');

// Register user
exports.registerUser = async (req, res) => {
  try {
    const { username, email, password, role } = req.body;

    // Check if user exists
    const existingUser = await User.findOne({ email });
    if (existingUser) {
      return res.status(400).json({ message: 'User already exists' });
    }

    // Hash password
    const salt = await bcrypt.genSalt(10);
    const hashedPassword = await bcrypt.hash(password, salt);

    // Create user
    const user = await User.create({
      username,
      email,
      password: hashedPassword,
      role: role || 'user'
    });

    // Generate token
    const token = jwt.sign(
      { id: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: process.env.JWT_EXPIRE }
    );

    res.status(201).json({
      success: true,
      token,
      user: {
        id: user._id,
        username: user.username,
        email: user.email,
        role: user.role
      }
    });
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
};

// Get all users (Admin only)
exports.getAllUsers = async (req, res) => {
  try {
    const users = await User.find().select('-password');
    res.json(users);
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
};

// Update user
exports.updateUser = async (req, res) => {
  try {
    const { id } = req.params;
    const updates = req.body;

    // Don't allow password updates through this endpoint
    delete updates.password;

    const user = await User.findByIdAndUpdate(
      id,
      updates,
      { new: true, runValidators: true }
    ).select('-password');

    if (!user) {
      return res.status(404).json({ message: 'User not found' });
    }

    res.json(user);
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
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
    res.status(500).json({ message: 'Server error', error: error.message });
  }
};
```

### Authentication Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

exports.protect = async (req, res, next) => {
  let token;

  if (req.headers.authorization && req.headers.authorization.startsWith('Bearer')) {
    token = req.headers.authorization.split(' ')[1];
  }

  if (!token) {
    return res.status(401).json({ message: 'Not authorized to access this route' });
  }

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = await User.findById(decoded.id).select('-password');
    next();
  } catch (error) {
    return res.status(401).json({ message: 'Not authorized to access this route' });
  }
};

exports.authorize = (...roles) => {
  return (req, res, next) => {
    if (!roles.includes(req.user.role)) {
      return res.status(403).json({
        message: `User role ${req.user.role} is not authorized to access this route`
      });
    }
    next();
  };
};
```

### Task Controller

```javascript
// backend/controllers/taskController.js
const Task = require('../models/Task');
const axios = require('axios');

exports.createTask = async (req, res) => {
  try {
    const { title, description, assignedTo, priority, dueDate } = req.body;

    const task = await Task.create({
      title,
      description,
      assignedTo,
      createdBy: req.user._id,
      priority,
      dueDate,
      status: 'todo'
    });

    res.status(201).json(task);
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
};

exports.getUserTasks = async (req, res) => {
  try {
    const { userId } = req.params;
    const tasks = await Task.find({ assignedTo: userId })
      .populate('assignedTo', 'username email')
      .populate('createdBy', 'username');
    
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
};

exports.updateTask = async (req, res) => {
  try {
    const { id } = req.params;
    const updates = req.body;

    const task = await Task.findByIdAndUpdate(
      id,
      updates,
      { new: true, runValidators: true }
    );

    if (!task) {
      return res.status(404).json({ message: 'Task not found' });
    }

    // If task completed, check for burnout risk
    if (updates.status === 'done' && task.assignedTo) {
      try {
        const userTasks = await Task.find({ assignedTo: task.assignedTo });
        const completedTasks = userTasks.filter(t => t.status === 'done');
        
        const mlResponse = await axios.post(
          `${process.env.ML_SERVICE_URL}/analyze/burnout`,
          {
            userId: task.assignedTo,
            workload: {
              tasksAssigned: userTasks.length,
              tasksCompleted: completedTasks.length,
              avgWorkHoursPerDay: 8 // Calculate from time tracking
            }
          }
        );

        if (mlResponse.data.burnoutRisk === 'high') {
          // Notify admin or create alert
          console.log('High burnout risk detected for user:', task.assignedTo);
        }
      } catch (mlError) {
        console.error('ML service error:', mlError.message);
      }
    }

    res.json(task);
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
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
from typing import List, Dict, Optional
import numpy as np
from sklearn.ensemble import RandomForestClassifier, IsolationForest
import joblib
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

# Models storage
models = {}

class RiskFeatures(BaseModel):
    userId: str
    features: Dict[str, any]

class BurnoutData(BaseModel):
    userId: str
    workload: Dict[str, any]

class AnomalyData(BaseModel):
    userId: str
    activity: Dict[str, any]

class TicketData(BaseModel):
    ticketId: str
    title: str
    description: str
    priority: str

class ProjectMetrics(BaseModel):
    projectId: str
    metrics: Dict[str, any]

# Initialize models
@app.on_event("startup")
async def load_models():
    model_path = os.getenv("MODEL_PATH", "./models")
    
    # Initialize or load risk prediction model
    risk_model_path = f"{model_path}/risk_model.pkl"
    if os.path.exists(risk_model_path):
        models['risk'] = joblib.load(risk_model_path)
    else:
        models['risk'] = RandomForestClassifier(n_estimators=100, random_state=42)
    
    # Initialize anomaly detection model
    models['anomaly'] = IsolationForest(contamination=0.1, random_state=42)
    
    print("ML models loaded successfully")

@app.get("/")
async def root():
    return {"message": "Enterprise User Management ML Service", "status": "running"}

@app.post("/predict/risk")
async def predict_risk(data: RiskFeatures):
    """
    Predict user risk score based on behavioral features
    """
    try:
        features = data.features
        
        # Extract and normalize features
        feature_vector = np.array([[
            features.get('loginFrequency', 0),
            features.get('taskCompletionRate', 0),
            features.get('avgTaskTime', 0),
            features.get('failedLoginAttempts', 0),
            len(features.get('dataAccessPatterns', []))
        ]])
        
        # Calculate risk score (simplified)
        risk_score = 0.0
        
        # Failed login attempts contribute to risk
        if features.get('failedLoginAttempts', 0) > 3:
            risk_score += 0.3
        
        # Low task completion rate increases risk
        if features.get('taskCompletionRate', 1.0) < 0.5:
            risk_score += 0.2
        
        # Unusual login patterns
        if features.get('loginFrequency', 0) > 20:
            risk_score += 0.2
        
        # Normalize to 0-1
        risk_score = min(risk_score, 1.0)
        
        # Determine risk level
        if risk_score < 0.3:
            risk_level = "low"
        elif risk_score < 0.7:
            risk_level = "medium"
        else:
            risk_level = "high"
        
        return {
            "userId": data.userId,
            "riskScore": round(risk_score, 2),
            "riskLevel": risk_level
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/detect/anomaly")
async def detect_anomaly(data: AnomalyData):
    """
    Detect anomalous user activity
    """
    try:
        activity = data.activity
        
        # Extract features from activity
        features = np.array([[
            hash(activity.get('action', '')) % 100,
            hash(activity.get('ipAddress', '')) % 100,
            hash(activity.get('deviceInfo', '')) % 100
        ]])
        
        # Use Isolation Forest for anomaly detection
        if not hasattr(models['anomaly'], 'estimators_'):
            # If model not trained, train on this data point (online learning simulation)
            models['anomaly'].fit(features)
            is_anomaly = False
            anomaly_score = 0.0
        else:
            prediction = models['anomaly'].predict(features)
