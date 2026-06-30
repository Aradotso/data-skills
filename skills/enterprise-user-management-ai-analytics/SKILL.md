---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for risk detection, task management, and organizational insights
triggers:
  - "set up enterprise user management system"
  - "implement AI analytics for user management"
  - "create user management dashboard with ML features"
  - "integrate AI ticket classification system"
  - "build task management with burnout detection"
  - "configure enterprise user system with authentication"
  - "add predictive analytics to user management"
  - "implement anomaly detection for user behavior"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System with AI Analytics is a full-stack JavaScript application that provides centralized user, task, and ticket management with integrated machine learning capabilities. The system offers JWT authentication, role-based access control, Kanban-style task tracking, and AI-powered features including risk prediction, anomaly detection, burnout analysis, and intelligent ticket routing.

**Architecture:**
- Frontend: React.js
- Backend: Node.js with REST APIs
- ML Service: FastAPI with scikit-learn and River
- Database: MongoDB
- Authentication: JWT tokens

## Installation

### Complete System Setup

```bash
# Clone the repository
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics

# Backend setup
cd backend
npm install
npm start
# Runs on http://localhost:5000

# ML service setup (in new terminal)
cd ml-service
pip install -r requirements.txt
uvicorn main:app --reload
# Runs on http://localhost:8000

# Frontend setup (in new terminal)
cd frontend
npm install
npm start
# Runs on http://localhost:3000
```

### Environment Configuration

Create `.env` files in each service:

**backend/.env:**
```env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_user_management
JWT_SECRET=your_jwt_secret_here
JWT_EXPIRE=24h
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
```

**ml-service/.env:**
```env
MODEL_PATH=./models
LOG_LEVEL=INFO
BACKEND_URL=http://localhost:5000
```

**frontend/.env:**
```env
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
```

## Core Features and Usage

### Authentication System

**User Login:**
```javascript
// frontend/src/services/authService.js
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL;

export const login = async (email, password) => {
  try {
    const response = await axios.post(`${API_URL}/api/auth/login`, {
      email,
      password
    });
    
    // Store JWT token
    localStorage.setItem('token', response.data.token);
    localStorage.setItem('user', JSON.stringify(response.data.user));
    
    return response.data;
  } catch (error) {
    throw error.response?.data?.message || 'Login failed';
  }
};

export const logout = () => {
  localStorage.removeItem('token');
  localStorage.removeItem('user');
};

export const getCurrentUser = () => {
  const user = localStorage.getItem('user');
  return user ? JSON.parse(user) : null;
};

export const getAuthHeader = () => {
  const token = localStorage.getItem('token');
  return token ? { Authorization: `Bearer ${token}` } : {};
};
```

**Backend Authentication Middleware:**
```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');

const authMiddleware = (req, res, next) => {
  try {
    const token = req.header('Authorization')?.replace('Bearer ', '');
    
    if (!token) {
      return res.status(401).json({ message: 'Access denied. No token provided.' });
    }
    
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch (error) {
    res.status(401).json({ message: 'Invalid token' });
  }
};

const adminOnly = (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ message: 'Access denied. Admin only.' });
  }
  next();
};

module.exports = { authMiddleware, adminOnly };
```

### User Management (Admin)

**User CRUD Operations:**
```javascript
// backend/routes/users.js
const express = require('express');
const router = express.Router();
const User = require('../models/User');
const { authMiddleware, adminOnly } = require('../middleware/auth');

// Get all users
router.get('/', authMiddleware, adminOnly, async (req, res) => {
  try {
    const users = await User.find().select('-password');
    res.json(users);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Create user
router.post('/', authMiddleware, adminOnly, async (req, res) => {
  try {
    const { name, email, role, department } = req.body;
    
    const existingUser = await User.findOne({ email });
    if (existingUser) {
      return res.status(400).json({ message: 'User already exists' });
    }
    
    const user = new User({
      name,
      email,
      role,
      department,
      password: 'default123' // Should be hashed
    });
    
    await user.save();
    res.status(201).json(user);
  } catch (error) {
    res.status(400).json({ message: error.message });
  }
});

// Update user
router.put('/:id', authMiddleware, adminOnly, async (req, res) => {
  try {
    const user = await User.findByIdAndUpdate(
      req.params.id,
      req.body,
      { new: true }
    ).select('-password');
    
    if (!user) {
      return res.status(404).json({ message: 'User not found' });
    }
    
    res.json(user);
  } catch (error) {
    res.status(400).json({ message: error.message });
  }
});

// Delete user
router.delete('/:id', authMiddleware, adminOnly, async (req, res) => {
  try {
    const user = await User.findByIdAndDelete(req.params.id);
    
    if (!user) {
      return res.status(404).json({ message: 'User not found' });
    }
    
    res.json({ message: 'User deleted successfully' });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

module.exports = router;
```

**Frontend User Management Component:**
```javascript
// frontend/src/components/Admin/UserManagement.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import { getAuthHeader } from '../../services/authService';

const UserManagement = () => {
  const [users, setUsers] = useState([]);
  const [formData, setFormData] = useState({
    name: '',
    email: '',
    role: 'user',
    department: ''
  });
  const [editingId, setEditingId] = useState(null);

  const API_URL = process.env.REACT_APP_API_URL;

  useEffect(() => {
    fetchUsers();
  }, []);

  const fetchUsers = async () => {
    try {
      const response = await axios.get(`${API_URL}/api/users`, {
        headers: getAuthHeader()
      });
      setUsers(response.data);
    } catch (error) {
      console.error('Error fetching users:', error);
    }
  };

  const handleSubmit = async (e) => {
    e.preventDefault();
    try {
      if (editingId) {
        await axios.put(
          `${API_URL}/api/users/${editingId}`,
          formData,
          { headers: getAuthHeader() }
        );
      } else {
        await axios.post(
          `${API_URL}/api/users`,
          formData,
          { headers: getAuthHeader() }
        );
      }
      
      setFormData({ name: '', email: '', role: 'user', department: '' });
      setEditingId(null);
      fetchUsers();
    } catch (error) {
      console.error('Error saving user:', error);
    }
  };

  const handleDelete = async (id) => {
    if (window.confirm('Are you sure you want to delete this user?')) {
      try {
        await axios.delete(`${API_URL}/api/users/${id}`, {
          headers: getAuthHeader()
        });
        fetchUsers();
      } catch (error) {
        console.error('Error deleting user:', error);
      }
    }
  };

  return (
    <div className="user-management">
      <h2>User Management</h2>
      
      <form onSubmit={handleSubmit}>
        <input
          type="text"
          placeholder="Name"
          value={formData.name}
          onChange={(e) => setFormData({ ...formData, name: e.target.value })}
          required
        />
        <input
          type="email"
          placeholder="Email"
          value={formData.email}
          onChange={(e) => setFormData({ ...formData, email: e.target.value })}
          required
        />
        <select
          value={formData.role}
          onChange={(e) => setFormData({ ...formData, role: e.target.value })}
        >
          <option value="user">User</option>
          <option value="admin">Admin</option>
        </select>
        <input
          type="text"
          placeholder="Department"
          value={formData.department}
          onChange={(e) => setFormData({ ...formData, department: e.target.value })}
        />
        <button type="submit">{editingId ? 'Update' : 'Add'} User</button>
      </form>

      <table>
        <thead>
          <tr>
            <th>Name</th>
            <th>Email</th>
            <th>Role</th>
            <th>Department</th>
            <th>Actions</th>
          </tr>
        </thead>
        <tbody>
          {users.map(user => (
            <tr key={user._id}>
              <td>{user.name}</td>
              <td>{user.email}</td>
              <td>{user.role}</td>
              <td>{user.department}</td>
              <td>
                <button onClick={() => {
                  setFormData(user);
                  setEditingId(user._id);
                }}>Edit</button>
                <button onClick={() => handleDelete(user._id)}>Delete</button>
              </td>
            </tr>
          ))}
        </tbody>
      </table>
    </div>
  );
};

export default UserManagement;
```

### Task Management System

**Task Model and Routes:**
```javascript
// backend/models/Task.js
const mongoose = require('mongoose');

const taskSchema = new mongoose.Schema({
  title: { type: String, required: true },
  description: String,
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  status: { 
    type: String, 
    enum: ['todo', 'inprogress', 'done'], 
    default: 'todo' 
  },
  priority: { 
    type: String, 
    enum: ['low', 'medium', 'high', 'urgent'], 
    default: 'medium' 
  },
  dueDate: Date,
  timeSpent: { type: Number, default: 0 }, // in minutes
  createdBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  createdAt: { type: Date, default: Date.now },
  updatedAt: { type: Date, default: Date.now }
});

module.exports = mongoose.model('Task', taskSchema);
```

```javascript
// backend/routes/tasks.js
const express = require('express');
const router = express.Router();
const Task = require('../models/Task');
const { authMiddleware } = require('../middleware/auth');

// Get user's tasks
router.get('/my-tasks', authMiddleware, async (req, res) => {
  try {
    const tasks = await Task.find({ assignedTo: req.user.id })
      .populate('assignedTo', 'name email')
      .populate('createdBy', 'name');
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Create task
router.post('/', authMiddleware, async (req, res) => {
  try {
    const task = new Task({
      ...req.body,
      createdBy: req.user.id
    });
    await task.save();
    res.status(201).json(task);
  } catch (error) {
    res.status(400).json({ message: error.message });
  }
});

// Update task status
router.patch('/:id/status', authMiddleware, async (req, res) => {
  try {
    const { status } = req.body;
    const task = await Task.findByIdAndUpdate(
      req.params.id,
      { status, updatedAt: Date.now() },
      { new: true }
    );
    res.json(task);
  } catch (error) {
    res.status(400).json({ message: error.message });
  }
});

// Track time on task
router.patch('/:id/time', authMiddleware, async (req, res) => {
  try {
    const { minutes } = req.body;
    const task = await Task.findByIdAndUpdate(
      req.params.id,
      { $inc: { timeSpent: minutes }, updatedAt: Date.now() },
      { new: true }
    );
    res.json(task);
  } catch (error) {
    res.status(400).json({ message: error.message });
  }
});

module.exports = router;
```

**Kanban Board Component:**
```javascript
// frontend/src/components/User/KanbanBoard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import { getAuthHeader } from '../../services/authService';

const KanbanBoard = () => {
  const [tasks, setTasks] = useState({ todo: [], inprogress: [], done: [] });
  const API_URL = process.env.REACT_APP_API_URL;

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    try {
      const response = await axios.get(`${API_URL}/api/tasks/my-tasks`, {
        headers: getAuthHeader()
      });
      
      const groupedTasks = {
        todo: response.data.filter(t => t.status === 'todo'),
        inprogress: response.data.filter(t => t.status === 'inprogress'),
        done: response.data.filter(t => t.status === 'done')
      };
      
      setTasks(groupedTasks);
    } catch (error) {
      console.error('Error fetching tasks:', error);
    }
  };

  const moveTask = async (taskId, newStatus) => {
    try {
      await axios.patch(
        `${API_URL}/api/tasks/${taskId}/status`,
        { status: newStatus },
        { headers: getAuthHeader() }
      );
      fetchTasks();
    } catch (error) {
      console.error('Error moving task:', error);
    }
  };

  const TaskCard = ({ task }) => (
    <div className="task-card" draggable onDragStart={(e) => {
      e.dataTransfer.setData('taskId', task._id);
      e.dataTransfer.setData('currentStatus', task.status);
    }}>
      <h4>{task.title}</h4>
      <p>{task.description}</p>
      <span className={`priority ${task.priority}`}>{task.priority}</span>
      {task.dueDate && <p>Due: {new Date(task.dueDate).toLocaleDateString()}</p>}
      <p>Time: {task.timeSpent} min</p>
    </div>
  );

  const Column = ({ title, status, tasks }) => (
    <div 
      className="kanban-column"
      onDragOver={(e) => e.preventDefault()}
      onDrop={(e) => {
        e.preventDefault();
        const taskId = e.dataTransfer.getData('taskId');
        const currentStatus = e.dataTransfer.getData('currentStatus');
        if (currentStatus !== status) {
          moveTask(taskId, status);
        }
      }}
    >
      <h3>{title}</h3>
      <div className="task-list">
        {tasks.map(task => <TaskCard key={task._id} task={task} />)}
      </div>
    </div>
  );

  return (
    <div className="kanban-board">
      <Column title="To Do" status="todo" tasks={tasks.todo} />
      <Column title="In Progress" status="inprogress" tasks={tasks.inprogress} />
      <Column title="Done" status="done" tasks={tasks.done} />
    </div>
  );
};

export default KanbanBoard;
```

### AI Integration - Ticket Classification

**ML Service API:**
```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import List, Optional
import numpy as np
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.naive_bayes import MultinomialNB
import joblib
import os

app = FastAPI()

# Load or initialize models
MODEL_PATH = os.getenv('MODEL_PATH', './models')
os.makedirs(MODEL_PATH, exist_ok=True)

class Ticket(BaseModel):
    title: str
    description: str
    priority: Optional[str] = 'medium'

class TicketClassificationResponse(BaseModel):
    category: str
    confidence: float
    suggested_assignee: Optional[str] = None

# Initialize classifier
try:
    vectorizer = joblib.load(f'{MODEL_PATH}/vectorizer.pkl')
    classifier = joblib.load(f'{MODEL_PATH}/ticket_classifier.pkl')
except:
    vectorizer = TfidfVectorizer(max_features=1000)
    classifier = MultinomialNB()
    # You would train with real data here

@app.post('/api/ml/classify-ticket', response_model=TicketClassificationResponse)
async def classify_ticket(ticket: Ticket):
    try:
        # Combine title and description
        text = f"{ticket.title} {ticket.description}"
        
        # Transform text
        features = vectorizer.transform([text])
        
        # Predict category
        category_codes = {
            0: 'Technical Support',
            1: 'Account Issues',
            2: 'Feature Request',
            3: 'Bug Report',
            4: 'General Inquiry'
        }
        
        prediction = classifier.predict(features)[0]
        confidence = classifier.predict_proba(features).max()
        
        category = category_codes.get(prediction, 'General Inquiry')
        
        # Route to appropriate team
        assignee_mapping = {
            'Technical Support': 'tech-team',
            'Bug Report': 'dev-team',
            'Account Issues': 'support-team',
            'Feature Request': 'product-team',
            'General Inquiry': 'support-team'
        }
        
        return TicketClassificationResponse(
            category=category,
            confidence=float(confidence),
            suggested_assignee=assignee_mapping.get(category)
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

class UserBehavior(BaseModel):
    user_id: str
    login_times: List[str]
    failed_logins: int
    data_access_count: int
    unusual_hours: int

class RiskAssessment(BaseModel):
    risk_score: float
    risk_level: str
    factors: List[str]

@app.post('/api/ml/assess-risk', response_model=RiskAssessment)
async def assess_risk(behavior: UserBehavior):
    try:
        # Calculate risk score based on behavior
        risk_score = 0.0
        factors = []
        
        if behavior.failed_logins > 5:
            risk_score += 0.3
            factors.append('Multiple failed login attempts')
        
        if behavior.unusual_hours > 10:
            risk_score += 0.25
            factors.append('Unusual working hours')
        
        if behavior.data_access_count > 100:
            risk_score += 0.2
            factors.append('High data access volume')
        
        # Normalize to 0-1
        risk_score = min(risk_score, 1.0)
        
        # Determine risk level
        if risk_score < 0.3:
            risk_level = 'Low'
        elif risk_score < 0.6:
            risk_level = 'Medium'
        else:
            risk_level = 'High'
        
        return RiskAssessment(
            risk_score=risk_score,
            risk_level=risk_level,
            factors=factors if factors else ['No risk factors detected']
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

class WorkloadData(BaseModel):
    user_id: str
    tasks_assigned: int
    tasks_completed: int
    hours_worked: float
    overtime_hours: float

class BurnoutAnalysis(BaseModel):
    burnout_risk: str
    score: float
    recommendations: List[str]

@app.post('/api/ml/analyze-burnout', response_model=BurnoutAnalysis)
async def analyze_burnout(workload: WorkloadData):
    try:
        score = 0.0
        recommendations = []
        
        # Calculate burnout indicators
        completion_rate = workload.tasks_completed / max(workload.tasks_assigned, 1)
        
        if workload.hours_worked > 45:
            score += 0.3
            recommendations.append('Reduce weekly working hours')
        
        if workload.overtime_hours > 10:
            score += 0.25
            recommendations.append('Limit overtime hours')
        
        if completion_rate < 0.6:
            score += 0.2
            recommendations.append('Review task assignment and priorities')
        
        if workload.tasks_assigned > 15:
            score += 0.25
            recommendations.append('Reduce task load')
        
        # Determine risk level
        if score < 0.3:
            risk = 'Low'
        elif score < 0.6:
            risk = 'Medium'
        else:
            risk = 'High'
        
        if not recommendations:
            recommendations = ['Workload appears balanced']
        
        return BurnoutAnalysis(
            burnout_risk=risk,
            score=min(score, 1.0),
            recommendations=recommendations
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

if __name__ == '__main__':
    import uvicorn
    uvicorn.run(app, host='0.0.0.0', port=8000)
```

**Backend Integration with ML Service:**
```javascript
// backend/services/mlService.js
const axios = require('axios');

const ML_API_URL = process.env.ML_SERVICE_URL || 'http://localhost:8000';

const classifyTicket = async (title, description, priority) => {
  try {
    const response = await axios.post(`${ML_API_URL}/api/ml/classify-ticket`, {
      title,
      description,
      priority
    });
    return response.data;
  } catch (error) {
    console.error('ML classification error:', error);
    return { category: 'General Inquiry', confidence: 0.5 };
  }
};

const assessUserRisk = async (userId, behaviorData) => {
  try {
    const response = await axios.post(`${ML_API_URL}/api/ml/assess-risk`, {
      user_id: userId,
      ...behaviorData
    });
    return response.data;
  } catch (error) {
    console.error('Risk assessment error:', error);
    return { risk_score: 0, risk_level: 'Low', factors: [] };
  }
};

const analyzeBurnout = async (userId, workloadData) => {
  try {
    const response = await axios.post(`${ML_API_URL}/api/ml/analyze-burnout`, {
      user_id: userId,
      ...workloadData
    });
    return response.data;
  } catch (error) {
    console.error('Burnout analysis error:', error);
    return { burnout_risk: 'Low', score: 0, recommendations: [] };
  }
};

module.exports = {
  classifyTicket,
  assessUserRisk,
  analyzeBurnout
};
```

### Support Ticket System with AI

```javascript
// backend/routes/tickets.js
const express = require('express');
const router = express.Router();
const Ticket = require('../models/Ticket');
const { authMiddleware } = require('../middleware/auth');
const { classifyTicket } = require('../services/mlService');

// Create ticket with AI classification
router.post('/', authMiddleware, async (req, res) => {
  try {
    const { title, description, priority } = req.body;
    
    // Get AI classification
    const classification = await classifyTicket(title, description, priority);
    
    const ticket = new Ticket({
      title,
      description,
      priority,
      category: classification.category,
      assignedTeam: classification.suggested_assignee,
      confidence: classification.confidence,
      createdBy: req.user.id,
      status: 'open'
    });
    
    await ticket.save();
    res.status(201).json(ticket);
  } catch (error) {
    res.status(400).json({ message: error.message });
  }
});

// Get user's tickets
router.get('/my-tickets', authMiddleware, async (req, res) => {
  try {
    const tickets = await Ticket.find({ createdBy: req.user.id })
      .sort({ createdAt: -1 });
    res.json(tickets);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Update ticket status
router.patch('/:id/status', authMiddleware, async (req, res) => {
  try {
    const { status } = req.body;
    const ticket = await Ticket.findByIdAndUpdate(
      req.params.id,
      { status, updatedAt: Date.now() },
      { new: true }
    );
    res.json(ticket);
  } catch (error) {
    res.status(400).json({ message: error.message });
  }
});

module.exports = router;
```

### Analytics Dashboard

**Admin Analytics API:**
```javascript
// backend/routes/analytics.js
const express = require('express');
const router = express.Router();
const User = require('../models/User');
const Task = require('../models/Task');
const Ticket = require('../models/Ticket');
const { authMiddleware, adminOnly } = require('../middleware/auth');
const { assessUserRisk, analyzeBurnout } = require('../services/mlService');

// Get organization overview
router.get('/overview', authMiddleware, adminOnly, async (req, res) => {
  try {
    const totalUsers = await User.countDocuments();
    const totalTasks = await Task.countDocuments();
    const totalTickets = await Ticket.countDocuments();
    
    const tasksByStatus = await Task.aggregate([
      { $group: { _id: '$status', count: { $sum: 1 } } }
    ]);
    
    const ticketsByStatus = await Ticket.aggregate([
      { $group: { _id: '$status', count: { $sum: 1 } } }
    ]);
    
    res.json({
      totalUsers,
      totalTasks,
      totalTickets,
      tasksByStatus,
      ticketsByStatus
    });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Get user productivity analysis
router.get('/productivity/:userId', authMiddleware, adminOnly, async (req, res) => {
  try {
    const userId = req.params.userId;
    
    const tasks = await Task.find({ assignedTo: userId });
    const completedTasks = tasks.filter(t => t.status === 'done').length;
    const totalTasks = tasks.length;
    const totalTimeSpent = tasks.reduce((sum, t) => sum + t.timeSpent, 0);
    
    // Calculate burnout risk
    const workloadData = {
      tasks_assigned: totalTasks,
      tasks_completed: completedTasks,
      hours_worked: totalTimeSpent / 60,
      overtime_hours: 0 // Calculate from time tracking
    };
    
    const burnoutAnalysis = await analyzeBurnout(userId, workloadData);
    
    res.json({
      tasksCompleted: completedTasks,
      totalTasks,
      completionRate: totalTasks > 0 ? completedTasks / totalTasks : 0,
      timeSpent: totalTimeSpent,
      burnoutAnalysis
    });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Get risk assessment for all users
router.get('/risk-assessment', authMiddleware, adminOnly, async (req, res) => {
  try {
    const users = await User.find().select('_id name email');
    const riskAssessments = [];
    
    for (const user of users) {
      // This would use real behavior tracking data
      const behaviorData = {
        login_times: [],
        failed_logins: 0,
        data_access_count: 0,
        unusual_hours: 0
      };
      
      const risk = await assessUserRisk(user._id.toString(), behaviorData);
      riskAssessments.push({
        user: user,
        ...risk
      });
    }
    
    res.json(riskAssessments);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

module.exports = router;
```

**Frontend Analytics Dashboard:**
```javascript
// frontend/src/components/Admin/Analytics.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import { getAuthHeader } from '../../services/auth
