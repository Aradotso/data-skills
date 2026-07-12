---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for risk detection, burnout analysis, and ticket classification
triggers:
  - "set up enterprise user management with AI analytics"
  - "implement user management system with anomaly detection"
  - "add AI-powered ticket classification"
  - "configure user dashboard with task tracking"
  - "integrate burnout detection and risk analysis"
  - "build admin panel with AI insights"
  - "create kanban board with time tracking"
  - "deploy enterprise user management system"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to help developers work with the Enterprise User Management System, a full-stack application that combines user/task management with AI-powered analytics for risk detection, anomaly detection, burnout analysis, and intelligent ticket routing.

## What This Project Does

The Enterprise User Management System provides:

- **User Management**: Role-based access control, user CRUD operations, authentication with JWT
- **Task Management**: Kanban board (To Do → In Progress → Done), time tracking, task assignment
- **Support System**: Ticket creation, tracking, and AI-based classification/routing
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, project delay prediction
- **Dashboards**: Separate admin and user interfaces with real-time insights

## Architecture

The system has three main components:

1. **Frontend** (React.js) - User interface on port 3000
2. **Backend** (Node.js/Express) - REST API on port 5000
3. **ML Service** (FastAPI + scikit-learn) - AI/ML endpoints on port 8000

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
```

Create `.env` file in `backend/`:

```env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise-user-management
JWT_SECRET=your_jwt_secret_key_here
JWT_EXPIRE=7d
NODE_ENV=development
ML_SERVICE_URL=http://localhost:8000
```

Start the backend:

```bash
npm start
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt
```

Create `.env` file in `ml-service/`:

```env
MODEL_PATH=./models
DATA_PATH=./data
LOG_LEVEL=INFO
```

Start the ML service:

```bash
uvicorn main:app --reload --port 8000
```

### Frontend Setup

```bash
cd frontend
npm install
```

Create `.env` file in `frontend/`:

```env
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_SERVICE_URL=http://localhost:8000
```

Start the frontend:

```bash
npm start
```

## Backend API Reference

### Authentication Endpoints

```javascript
// Register new user
POST /api/auth/register
{
  "name": "John Doe",
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

// Response includes JWT token
{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "user": {
    "id": "507f1f77bcf86cd799439011",
    "name": "John Doe",
    "email": "john@example.com",
    "role": "user"
  }
}
```

### User Management (Admin Only)

```javascript
// Get all users
GET /api/users
Headers: { "Authorization": "Bearer <token>" }

// Get user by ID
GET /api/users/:id

// Update user
PUT /api/users/:id
{
  "name": "Jane Doe",
  "role": "admin",
  "status": "active"
}

// Delete user
DELETE /api/users/:id
```

### Task Management

```javascript
// Create task
POST /api/tasks
{
  "title": "Implement user authentication",
  "description": "Add JWT-based authentication",
  "assignedTo": "507f1f77bcf86cd799439011",
  "priority": "high", // low, medium, high
  "dueDate": "2026-05-01T00:00:00.000Z",
  "status": "todo" // todo, inprogress, done
}

// Update task status
PATCH /api/tasks/:id/status
{
  "status": "inprogress"
}

// Get user tasks
GET /api/tasks/user/:userId

// Start time tracking
POST /api/tasks/:id/time/start

// Stop time tracking
POST /api/tasks/:id/time/stop
```

### Support Tickets

```javascript
// Create ticket
POST /api/tickets
{
  "title": "Cannot access dashboard",
  "description": "Getting 403 error when accessing admin panel",
  "priority": "high",
  "category": "technical" // technical, billing, general
}

// Get all tickets (admin)
GET /api/tickets

// Get user tickets
GET /api/tickets/user/:userId

// Update ticket
PATCH /api/tickets/:id
{
  "status": "in-progress",
  "assignedTo": "507f1f77bcf86cd799439011"
}
```

## ML Service API Reference

### Ticket Classification

```javascript
// Classify ticket using AI
POST http://localhost:8000/classify-ticket
{
  "title": "Cannot access dashboard",
  "description": "Getting 403 error when accessing admin panel"
}

// Response
{
  "category": "technical",
  "priority": "high",
  "confidence": 0.92,
  "suggested_department": "IT Support"
}
```

### Risk Detection

```javascript
// Analyze user risk
POST http://localhost:8000/analyze-risk
{
  "user_id": "507f1f77bcf86cd799439011",
  "login_attempts": 3,
  "failed_logins": 2,
  "unusual_hours": true,
  "location_changes": 5,
  "data_access_volume": 1500
}

// Response
{
  "risk_score": 0.78,
  "risk_level": "high", // low, medium, high
  "factors": [
    "Multiple failed login attempts",
    "Unusual access hours",
    "High location changes"
  ],
  "recommendation": "Require additional authentication"
}
```

### Anomaly Detection

```javascript
// Detect anomalies in user behavior
POST http://localhost:8000/detect-anomaly
{
  "user_id": "507f1f77bcf86cd799439011",
  "features": {
    "login_time": 3, // 3 AM
    "session_duration": 480, // minutes
    "files_accessed": 250,
    "api_calls": 5000,
    "data_downloaded": 5000 // MB
  }
}

// Response
{
  "is_anomaly": true,
  "anomaly_score": 0.85,
  "anomalous_features": [
    "unusual_login_time",
    "excessive_data_download",
    "high_api_calls"
  ],
  "severity": "critical"
}
```

### Burnout Detection

```javascript
// Analyze employee burnout risk
POST http://localhost:8000/detect-burnout
{
  "user_id": "507f1f77bcf86cd799439011",
  "tasks_completed": 45,
  "tasks_overdue": 8,
  "avg_work_hours": 11.5,
  "weekend_work": true,
  "missed_deadlines": 6,
  "stress_indicators": 7
}

// Response
{
  "burnout_risk": "high",
  "burnout_score": 0.82,
  "factors": [
    "Excessive work hours",
    "High missed deadlines",
    "Weekend work patterns"
  ],
  "recommendations": [
    "Redistribute workload",
    "Encourage time off",
    "Schedule wellness check-in"
  ]
}
```

### Project Delay Prediction

```javascript
// Predict project delays
POST http://localhost:8000/predict-delay
{
  "project_id": "proj_123",
  "planned_tasks": 50,
  "completed_tasks": 15,
  "days_elapsed": 30,
  "total_days": 90,
  "team_size": 5,
  "blockers": 3,
  "avg_velocity": 0.5
}

// Response
{
  "delay_prediction": true,
  "estimated_delay_days": 15,
  "completion_probability": 0.65,
  "risk_factors": [
    "Below target velocity",
    "Multiple blockers",
    "Insufficient progress"
  ],
  "mitigation_strategies": [
    "Increase team size",
    "Resolve blockers urgently",
    "Re-scope non-critical features"
  ]
}
```

## Frontend Integration Patterns

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
      const res = await axios.get(`${API_URL}/auth/me`);
      setUser(res.data.user);
    } catch (error) {
      localStorage.removeItem('token');
      delete axios.defaults.headers.common['Authorization'];
    } finally {
      setLoading(false);
    }
  };

  const login = async (email, password) => {
    const res = await axios.post(`${API_URL}/auth/login`, { email, password });
    localStorage.setItem('token', res.data.token);
    axios.defaults.headers.common['Authorization'] = `Bearer ${res.data.token}`;
    setUser(res.data.user);
    return res.data;
  };

  const logout = () => {
    localStorage.removeItem('token');
    delete axios.defaults.headers.common['Authorization'];
    setUser(null);
  };

  return { user, loading, login, logout };
};
```

### Task Management Component

```javascript
// src/components/KanbanBoard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL;

const KanbanBoard = ({ userId }) => {
  const [tasks, setTasks] = useState({ todo: [], inprogress: [], done: [] });

  useEffect(() => {
    fetchTasks();
  }, [userId]);

  const fetchTasks = async () => {
    try {
      const res = await axios.get(`${API_URL}/tasks/user/${userId}`);
      const grouped = {
        todo: res.data.filter(t => t.status === 'todo'),
        inprogress: res.data.filter(t => t.status === 'inprogress'),
        done: res.data.filter(t => t.status === 'done')
      };
      setTasks(grouped);
    } catch (error) {
      console.error('Error fetching tasks:', error);
    }
  };

  const moveTask = async (taskId, newStatus) => {
    try {
      await axios.patch(`${API_URL}/tasks/${taskId}/status`, {
        status: newStatus
      });
      fetchTasks();
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  const TaskCard = ({ task }) => (
    <div className="task-card" draggable>
      <h4>{task.title}</h4>
      <p>{task.description}</p>
      <span className={`priority-${task.priority}`}>{task.priority}</span>
      <div className="task-actions">
        {task.status !== 'done' && (
          <button onClick={() => moveTask(task._id, 
            task.status === 'todo' ? 'inprogress' : 'done')}>
            Move →
          </button>
        )}
      </div>
    </div>
  );

  return (
    <div className="kanban-board">
      <div className="kanban-column">
        <h3>To Do ({tasks.todo.length})</h3>
        {tasks.todo.map(task => <TaskCard key={task._id} task={task} />)}
      </div>
      <div className="kanban-column">
        <h3>In Progress ({tasks.inprogress.length})</h3>
        {tasks.inprogress.map(task => <TaskCard key={task._id} task={task} />)}
      </div>
      <div className="kanban-column">
        <h3>Done ({tasks.done.length})</h3>
        {tasks.done.map(task => <TaskCard key={task._id} task={task} />)}
      </div>
    </div>
  );
};

export default KanbanBoard;
```

### AI Analytics Integration

```javascript
// src/components/AIInsights.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const ML_URL = process.env.REACT_APP_ML_SERVICE_URL;

const AIInsights = ({ userId }) => {
  const [insights, setInsights] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchAIInsights();
  }, [userId]);

  const fetchAIInsights = async () => {
    try {
      // Fetch user data first
      const userRes = await axios.get(`${process.env.REACT_APP_API_URL}/users/${userId}/stats`);
      const stats = userRes.data;

      // Get AI predictions
      const [riskRes, burnoutRes] = await Promise.all([
        axios.post(`${ML_URL}/analyze-risk`, {
          user_id: userId,
          login_attempts: stats.loginAttempts,
          failed_logins: stats.failedLogins,
          unusual_hours: stats.unusualHours,
          location_changes: stats.locationChanges,
          data_access_volume: stats.dataAccess
        }),
        axios.post(`${ML_URL}/detect-burnout`, {
          user_id: userId,
          tasks_completed: stats.tasksCompleted,
          tasks_overdue: stats.tasksOverdue,
          avg_work_hours: stats.avgWorkHours,
          weekend_work: stats.weekendWork,
          missed_deadlines: stats.missedDeadlines,
          stress_indicators: stats.stressIndicators
        })
      ]);

      setInsights({
        risk: riskRes.data,
        burnout: burnoutRes.data
      });
    } catch (error) {
      console.error('Error fetching AI insights:', error);
    } finally {
      setLoading(false);
    }
  };

  if (loading) return <div>Loading AI insights...</div>;

  return (
    <div className="ai-insights">
      <div className="insight-card risk">
        <h3>Security Risk Analysis</h3>
        <div className={`risk-level ${insights.risk.risk_level}`}>
          {insights.risk.risk_level.toUpperCase()}
        </div>
        <p>Risk Score: {(insights.risk.risk_score * 100).toFixed(1)}%</p>
        <ul>
          {insights.risk.factors.map((factor, i) => (
            <li key={i}>{factor}</li>
          ))}
        </ul>
        <p className="recommendation">{insights.risk.recommendation}</p>
      </div>

      <div className="insight-card burnout">
        <h3>Burnout Risk Analysis</h3>
        <div className={`burnout-level ${insights.burnout.burnout_risk}`}>
          {insights.burnout.burnout_risk.toUpperCase()}
        </div>
        <p>Burnout Score: {(insights.burnout.burnout_score * 100).toFixed(1)}%</p>
        <h4>Contributing Factors:</h4>
        <ul>
          {insights.burnout.factors.map((factor, i) => (
            <li key={i}>{factor}</li>
          ))}
        </ul>
        <h4>Recommendations:</h4>
        <ul>
          {insights.burnout.recommendations.map((rec, i) => (
            <li key={i}>{rec}</li>
          ))}
        </ul>
      </div>
    </div>
  );
};

export default AIInsights;
```

### Smart Ticket Creation

```javascript
// src/components/CreateTicket.jsx
import React, { useState } from 'react';
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL;
const ML_URL = process.env.REACT_APP_ML_SERVICE_URL;

const CreateTicket = () => {
  const [formData, setFormData] = useState({
    title: '',
    description: ''
  });
  const [aiSuggestions, setAiSuggestions] = useState(null);

  const handleChange = (e) => {
    setFormData({ ...formData, [e.target.name]: e.target.value });
  };

  const getAISuggestions = async () => {
    if (!formData.title || !formData.description) return;

    try {
      const res = await axios.post(`${ML_URL}/classify-ticket`, {
        title: formData.title,
        description: formData.description
      });
      setAiSuggestions(res.data);
    } catch (error) {
      console.error('Error getting AI suggestions:', error);
    }
  };

  const handleSubmit = async (e) => {
    e.preventDefault();
    
    try {
      const ticketData = {
        ...formData,
        category: aiSuggestions?.category || 'general',
        priority: aiSuggestions?.priority || 'medium'
      };

      await axios.post(`${API_URL}/tickets`, ticketData);
      alert('Ticket created successfully!');
      setFormData({ title: '', description: '' });
      setAiSuggestions(null);
    } catch (error) {
      console.error('Error creating ticket:', error);
      alert('Failed to create ticket');
    }
  };

  return (
    <div className="create-ticket">
      <h2>Create Support Ticket</h2>
      <form onSubmit={handleSubmit}>
        <input
          type="text"
          name="title"
          placeholder="Ticket Title"
          value={formData.title}
          onChange={handleChange}
          required
        />
        <textarea
          name="description"
          placeholder="Describe your issue..."
          value={formData.description}
          onChange={handleChange}
          rows="5"
          required
        />
        <button type="button" onClick={getAISuggestions}>
          Get AI Suggestions
        </button>

        {aiSuggestions && (
          <div className="ai-suggestions">
            <h4>AI Recommendations:</h4>
            <p>Category: <strong>{aiSuggestions.category}</strong></p>
            <p>Priority: <strong>{aiSuggestions.priority}</strong></p>
            <p>Department: <strong>{aiSuggestions.suggested_department}</strong></p>
            <p>Confidence: {(aiSuggestions.confidence * 100).toFixed(1)}%</p>
          </div>
        )}

        <button type="submit">Submit Ticket</button>
      </form>
    </div>
  );
};

export default CreateTicket;
```

## Database Models

### User Schema (MongoDB)

```javascript
// backend/models/User.js
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');

const userSchema = new mongoose.Schema({
  name: {
    type: String,
    required: true
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
    select: false
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
  department: String,
  position: String,
  loginAttempts: {
    type: Number,
    default: 0
  },
  lastLogin: Date,
  createdAt: {
    type: Date,
    default: Date.now
  }
});

// Hash password before saving
userSchema.pre('save', async function(next) {
  if (!this.isModified('password')) return next();
  this.password = await bcrypt.hash(this.password, 12);
  next();
});

// Compare password method
userSchema.methods.comparePassword = async function(candidatePassword) {
  return await bcrypt.compare(candidatePassword, this.password);
};

module.exports = mongoose.model('User', userSchema);
```

### Task Schema

```javascript
// backend/models/Task.js
const mongoose = require('mongoose');

const taskSchema = new mongoose.Schema({
  title: {
    type: String,
    required: true
  },
  description: String,
  assignedTo: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User',
    required: true
  },
  assignedBy: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User'
  },
  status: {
    type: String,
    enum: ['todo', 'inprogress', 'done'],
    default: 'todo'
  },
  priority: {
    type: String,
    enum: ['low', 'medium', 'high'],
    default: 'medium'
  },
  dueDate: Date,
  timeTracking: [{
    startTime: Date,
    endTime: Date,
    duration: Number // in minutes
  }],
  totalTimeSpent: {
    type: Number,
    default: 0 // in minutes
  },
  createdAt: {
    type: Date,
    default: Date.now
  },
  completedAt: Date
});

module.exports = mongoose.model('Task', taskSchema);
```

### Ticket Schema

```javascript
// backend/models/Ticket.js
const mongoose = require('mongoose');

const ticketSchema = new mongoose.Schema({
  title: {
    type: String,
    required: true
  },
  description: {
    type: String,
    required: true
  },
  createdBy: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User',
    required: true
  },
  assignedTo: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User'
  },
  category: {
    type: String,
    enum: ['technical', 'billing', 'general', 'hr', 'security'],
    default: 'general'
  },
  priority: {
    type: String,
    enum: ['low', 'medium', 'high', 'critical'],
    default: 'medium'
  },
  status: {
    type: String,
    enum: ['open', 'in-progress', 'resolved', 'closed'],
    default: 'open'
  },
  aiClassification: {
    category: String,
    priority: String,
    confidence: Number,
    department: String
  },
  comments: [{
    user: {
      type: mongoose.Schema.Types.ObjectId,
      ref: 'User'
    },
    text: String,
    createdAt: {
      type: Date,
      default: Date.now
    }
  }],
  createdAt: {
    type: Date,
    default: Date.now
  },
  resolvedAt: Date
});

module.exports = mongoose.model('Ticket', ticketSchema);
```

## Configuration

### Backend Environment Variables

```env
# Server
PORT=5000
NODE_ENV=development

# Database
MONGODB_URI=mongodb://localhost:27017/enterprise-user-management
# Or use MongoDB Atlas
# MONGODB_URI=mongodb+srv://username:password@cluster.mongodb.net/dbname

# JWT
JWT_SECRET=your_very_secure_secret_key_change_in_production
JWT_EXPIRE=7d

# ML Service
ML_SERVICE_URL=http://localhost:8000

# Email (optional, for notifications)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASS=your_app_password

# Rate Limiting
MAX_LOGIN_ATTEMPTS=5
LOCKOUT_DURATION=15 # minutes
```

### ML Service Configuration

```python
# ml-service/config.py
import os
from pathlib import Path

class Config:
    # Paths
    BASE_DIR = Path(__file__).parent
    MODEL_PATH = os.getenv('MODEL_PATH', BASE_DIR / 'models')
    DATA_PATH = os.getenv('DATA_PATH', BASE_DIR / 'data')
    
    # ML Settings
    RISK_THRESHOLD = 0.7
    ANOMALY_THRESHOLD = 0.75
    BURNOUT_THRESHOLD = 0.65
    
    # Model Parameters
    RANDOM_STATE = 42
    TEST_SIZE = 0.2
    
    # Logging
    LOG_LEVEL = os.getenv('LOG_LEVEL', 'INFO')
    
    # API Settings
    CORS_ORIGINS = ['http://localhost:3000', 'http://localhost:5000']
    
config = Config()
```

## Common Patterns

### Protected Routes (Admin Only)

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

exports.protect = async (req, res, next) => {
  let token;

  if (req.headers.authorization?.startsWith('Bearer')) {
    token = req.headers.authorization.split(' ')[1];
  }

  if (!token) {
    return res.status(401).json({ message: 'Not authorized' });
  }

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = await User.findById(decoded.id);
    next();
  } catch (error) {
    return res.status(401).json({ message: 'Token invalid' });
  }
};

exports.authorize = (...roles) => {
  return (req, res, next) => {
    if (!roles.includes(req.user.role)) {
      return res.status(403).json({
        message: 'Forbidden: Insufficient permissions'
      });
    }
    next();
  };
};
```

### Usage in Routes

```javascript
// backend/routes/users.js
const express = require('express');
const router = express.Router();
const { protect, authorize } = require('../middleware/auth');
const User = require('../models/User');

// Admin only routes
router.get('/', protect, authorize('admin'), async (req, res) => {
  const users = await User.find();
  res.json(users);
});

router.delete('/:id', protect, authorize('admin'), async (req, res) => {
  await User.findByIdAndDelete(req.params.id);
  res.json({ message: 'User deleted' });
});

module.exports = router;
```

### Time Tracking Implementation

```javascript
// backend/controllers/taskController.js
const Task = require('../models/Task');

exports.startTimeTracking = async (req, res) => {
  try {
    const task = await Task.findById(req.params.id);
    
    if (!task) {
      return res.status(404).json({ message: 'Task not found' });
    }

    // Check if already tracking
    const activeTracking = task.timeTracking.find(t => !t.endTime);
    if (activeTracking) {
      return res.status(400).json({ message: 'Time tracking already active' });
    }

    task.timeTracking.push({ startTime: new Date() });
    await task.save();

    res.json({ message: 'Time tracking started', task });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
};

exports.stopTimeTracking = async (req, res) => {
  try {
    const task = await Task.findById(req.params.id);
    
    if (!task) {
      return res.status(404).json({ message: 'Task not found' });
    }

    const activeTracking = task.timeTracking.find(t => !t.endTime);
    if (!activeTracking) {
      return res.status(400).json({ message: 'No active time tracking' });
    }

    activeTracking.endTime = new Date();
    activeTracking.duration = Math.round(
      (activeTracking.endTime - activeTracking.startTime) / 60000
    ); // Convert to minutes

    task.totalTimeSpent += activeTracking.duration;
    await task.save();

    res.json({ 
      message: 'Time tracking stopped', 
      duration: activeTracking.duration,
      task 
    });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
};
```

### ML Model Training Script

```python
# ml-service/train_models.py
import numpy as np
import pandas as pd
from sklearn.ensemble import RandomForestClassifier
from sklearn.preprocessing import StandardScaler
import joblib
from pathlib import Path

def train_risk_model():
    """Train the risk detection model"""
    # Sample training data structure
    # In production, load from database or CSV
    data = {
        'login_attempts': [1, 2, 5, 10, 3, 7, 15],
        'failed_logins': [0, 1, 3, 8, 1, 5, 12],
        'unusual_hours': [0, 0, 1, 1, 0, 1, 1],
        'location_changes': [1, 2, 5, 10, 2, 7, 15],
        'data_access_volume': [100, 200, 500, 2000, 150, 1000, 5000],
        'risk_label': [0, 0, 1, 1, 
