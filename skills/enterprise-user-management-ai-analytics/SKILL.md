---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics for task management, ticket routing, and risk detection
triggers:
  - "help me set up the enterprise user management system"
  - "how do I integrate AI analytics into user management"
  - "configure JWT authentication for user management app"
  - "implement AI ticket classification and routing"
  - "set up anomaly detection for user behavior"
  - "create a kanban board with task tracking"
  - "build admin dashboard with user analytics"
  - "integrate burnout detection and risk prediction"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

The Enterprise User Management System is a full-stack JavaScript application that combines user/task management with AI-powered analytics. It provides role-based access control, task tracking with Kanban boards, support ticket management, and machine learning features including risk prediction, anomaly detection, burnout analysis, and automated ticket classification.

**Key Components:**
- **Frontend**: React.js application with admin and user dashboards
- **Backend**: Node.js REST API with JWT authentication
- **ML Service**: FastAPI service using scikit-learn and River for real-time ML
- **Database**: MongoDB for persistent storage

## Installation

### Prerequisites
```bash
# Required software
node --version  # v14+ required
python --version  # Python 3.8+ required
mongod --version  # MongoDB 4.4+ required
```

### Full System Setup

```bash
# Clone repository
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics

# Backend setup
cd backend
npm install
cp .env.example .env  # Configure environment variables
npm start  # Runs on http://localhost:5000

# ML Service setup (new terminal)
cd ml-service
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate
pip install -r requirements.txt
uvicorn main:app --reload  # Runs on http://localhost:8000

# Frontend setup (new terminal)
cd frontend
npm install
npm start  # Runs on http://localhost:3000
```

### Environment Configuration

**Backend (.env)**
```bash
# Server
PORT=5000
NODE_ENV=development

# Database
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt

# Authentication
JWT_SECRET=your_secure_random_string_here
JWT_EXPIRE=7d

# ML Service
ML_SERVICE_URL=http://localhost:8000

# CORS
FRONTEND_URL=http://localhost:3000
```

**ML Service (.env)**
```bash
# API Configuration
API_HOST=0.0.0.0
API_PORT=8000

# Model Configuration
MODEL_PATH=./models
ENABLE_ONLINE_LEARNING=true

# Logging
LOG_LEVEL=INFO
```

## Backend API Usage

### Authentication

```javascript
// Register new user
const registerUser = async (userData) => {
  const response = await fetch('http://localhost:5000/api/auth/register', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      username: userData.username,
      email: userData.email,
      password: userData.password,
      role: userData.role || 'user'  // 'user' or 'admin'
    })
  });
  return await response.json();
};

// Login
const loginUser = async (email, password) => {
  const response = await fetch('http://localhost:5000/api/auth/login', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ email, password })
  });
  const data = await response.json();
  // Store JWT token
  localStorage.setItem('token', data.token);
  return data;
};

// Authenticated request helper
const authenticatedFetch = async (url, options = {}) => {
  const token = localStorage.getItem('token');
  return fetch(url, {
    ...options,
    headers: {
      ...options.headers,
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    }
  });
};
```

### User Management (Admin)

```javascript
// Get all users
const getAllUsers = async () => {
  const response = await authenticatedFetch('http://localhost:5000/api/users');
  return await response.json();
};

// Create user
const createUser = async (userData) => {
  const response = await authenticatedFetch('http://localhost:5000/api/users', {
    method: 'POST',
    body: JSON.stringify({
      username: userData.username,
      email: userData.email,
      role: userData.role,
      department: userData.department
    })
  });
  return await response.json();
};

// Update user
const updateUser = async (userId, updates) => {
  const response = await authenticatedFetch(`http://localhost:5000/api/users/${userId}`, {
    method: 'PUT',
    body: JSON.stringify(updates)
  });
  return await response.json();
};

// Delete user
const deleteUser = async (userId) => {
  await authenticatedFetch(`http://localhost:5000/api/users/${userId}`, {
    method: 'DELETE'
  });
};
```

### Task Management

```javascript
// Create task
const createTask = async (taskData) => {
  const response = await authenticatedFetch('http://localhost:5000/api/tasks', {
    method: 'POST',
    body: JSON.stringify({
      title: taskData.title,
      description: taskData.description,
      assignedTo: taskData.userId,
      priority: taskData.priority,  // 'low', 'medium', 'high'
      status: 'todo',  // 'todo', 'in-progress', 'done'
      dueDate: taskData.dueDate
    })
  });
  return await response.json();
};

// Get user tasks
const getUserTasks = async (userId) => {
  const response = await authenticatedFetch(`http://localhost:5000/api/tasks/user/${userId}`);
  return await response.json();
};

// Update task status
const updateTaskStatus = async (taskId, status) => {
  const response = await authenticatedFetch(`http://localhost:5000/api/tasks/${taskId}`, {
    method: 'PATCH',
    body: JSON.stringify({ status })
  });
  return await response.json();
};

// Track time on task
const trackTime = async (taskId, timeSpent) => {
  const response = await authenticatedFetch(`http://localhost:5000/api/tasks/${taskId}/time`, {
    method: 'POST',
    body: JSON.stringify({ 
      timeSpent,  // in minutes
      timestamp: new Date().toISOString()
    })
  });
  return await response.json();
};
```

### Support Tickets

```javascript
// Create support ticket
const createTicket = async (ticketData) => {
  const response = await authenticatedFetch('http://localhost:5000/api/tickets', {
    method: 'POST',
    body: JSON.stringify({
      subject: ticketData.subject,
      description: ticketData.description,
      category: ticketData.category,  // 'technical', 'billing', 'general'
      priority: ticketData.priority
    })
  });
  return await response.json();
};

// Get user tickets
const getUserTickets = async () => {
  const response = await authenticatedFetch('http://localhost:5000/api/tickets/user');
  return await response.json();
};

// Update ticket status
const updateTicketStatus = async (ticketId, status) => {
  const response = await authenticatedFetch(`http://localhost:5000/api/tickets/${ticketId}`, {
    method: 'PATCH',
    body: JSON.stringify({ status })  // 'open', 'in-progress', 'resolved', 'closed'
  });
  return await response.json();
};
```

## ML Service API Usage

### AI-Powered Ticket Classification

```javascript
// Classify ticket and auto-assign
const classifyTicket = async (ticketText) => {
  const response = await fetch('http://localhost:8000/api/ml/classify-ticket', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      text: ticketText,
      subject: "Subject line of ticket"
    })
  });
  const result = await response.json();
  // Returns: { category: 'technical', priority: 'high', suggested_assignee: 'dept_id' }
  return result;
};
```

### Risk Detection

```javascript
// Detect user behavior risks
const detectRisk = async (userId, behaviorData) => {
  const response = await fetch('http://localhost:8000/api/ml/risk-detection', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      user_id: userId,
      login_attempts: behaviorData.loginAttempts,
      failed_logins: behaviorData.failedLogins,
      access_patterns: behaviorData.accessPatterns,
      data_access_volume: behaviorData.dataAccessVolume,
      unusual_hours: behaviorData.unusualHours
    })
  });
  const result = await response.json();
  // Returns: { risk_score: 0.75, risk_level: 'high', factors: [...] }
  return result;
};
```

### Anomaly Detection

```javascript
// Detect anomalous user activity
const detectAnomaly = async (userActivity) => {
  const response = await fetch('http://localhost:8000/api/ml/anomaly-detection', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      user_id: userActivity.userId,
      features: {
        login_time: userActivity.loginTime,
        session_duration: userActivity.sessionDuration,
        pages_visited: userActivity.pagesVisited,
        api_calls: userActivity.apiCalls,
        data_downloaded: userActivity.dataDownloaded
      }
    })
  });
  const result = await response.json();
  // Returns: { is_anomaly: true, anomaly_score: 0.85, details: {...} }
  return result;
};
```

### Burnout Detection

```javascript
// Analyze employee burnout risk
const detectBurnout = async (userId) => {
  const response = await fetch('http://localhost:8000/api/ml/burnout-detection', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      user_id: userId,
      workload_metrics: {
        tasks_assigned: 25,
        tasks_completed: 18,
        avg_working_hours: 9.5,
        overtime_hours: 15,
        missed_deadlines: 3,
        ticket_response_time: 120  // minutes
      }
    })
  });
  const result = await response.json();
  // Returns: { burnout_risk: 'high', score: 0.78, recommendations: [...] }
  return result;
};
```

### Predictive Project Insights

```javascript
// Predict project delays
const predictProjectDelay = async (projectData) => {
  const response = await fetch('http://localhost:8000/api/ml/project-insights', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      project_id: projectData.id,
      total_tasks: projectData.totalTasks,
      completed_tasks: projectData.completedTasks,
      team_size: projectData.teamSize,
      avg_task_completion_time: projectData.avgCompletionTime,
      deadline: projectData.deadline,
      current_velocity: projectData.velocity
    })
  });
  const result = await response.json();
  // Returns: { delay_probability: 0.65, estimated_completion: '2026-05-15', risks: [...] }
  return result;
};
```

## Frontend React Components

### Protected Route Component

```javascript
import React from 'react';
import { Navigate } from 'react-router-dom';

const ProtectedRoute = ({ children, requiredRole }) => {
  const token = localStorage.getItem('token');
  const userRole = localStorage.getItem('userRole');

  if (!token) {
    return <Navigate to="/login" />;
  }

  if (requiredRole && userRole !== requiredRole) {
    return <Navigate to="/unauthorized" />;
  }

  return children;
};

export default ProtectedRoute;
```

### Kanban Board Component

```javascript
import React, { useState, useEffect } from 'react';
import './KanbanBoard.css';

const KanbanBoard = ({ userId }) => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });

  useEffect(() => {
    fetchTasks();
  }, [userId]);

  const fetchTasks = async () => {
    const response = await authenticatedFetch(`http://localhost:5000/api/tasks/user/${userId}`);
    const data = await response.json();
    
    setTasks({
      todo: data.filter(t => t.status === 'todo'),
      inProgress: data.filter(t => t.status === 'in-progress'),
      done: data.filter(t => t.status === 'done')
    });
  };

  const moveTask = async (taskId, newStatus) => {
    await updateTaskStatus(taskId, newStatus);
    fetchTasks();
  };

  const TaskCard = ({ task }) => (
    <div className="task-card" draggable onDragStart={(e) => e.dataTransfer.setData('taskId', task._id)}>
      <h4>{task.title}</h4>
      <p>{task.description}</p>
      <span className={`priority-${task.priority}`}>{task.priority}</span>
      <div className="task-actions">
        {task.status !== 'done' && (
          <button onClick={() => moveTask(task._id, task.status === 'todo' ? 'in-progress' : 'done')}>
            Move →
          </button>
        )}
      </div>
    </div>
  );

  const Column = ({ title, status, tasks }) => (
    <div className="kanban-column" onDrop={(e) => {
      e.preventDefault();
      const taskId = e.dataTransfer.getData('taskId');
      moveTask(taskId, status);
    }} onDragOver={(e) => e.preventDefault()}>
      <h3>{title} ({tasks.length})</h3>
      {tasks.map(task => <TaskCard key={task._id} task={task} />)}
    </div>
  );

  return (
    <div className="kanban-board">
      <Column title="To Do" status="todo" tasks={tasks.todo} />
      <Column title="In Progress" status="in-progress" tasks={tasks.inProgress} />
      <Column title="Done" status="done" tasks={tasks.done} />
    </div>
  );
};

export default KanbanBoard;
```

### AI-Powered Ticket Form

```javascript
import React, { useState } from 'react';

const TicketForm = ({ onSubmit }) => {
  const [formData, setFormData] = useState({
    subject: '',
    description: '',
    category: '',
    priority: ''
  });
  const [aiSuggestions, setAiSuggestions] = useState(null);

  const handleAIClassification = async () => {
    const classification = await classifyTicket(formData.description);
    setAiSuggestions(classification);
    setFormData(prev => ({
      ...prev,
      category: classification.category,
      priority: classification.priority
    }));
  };

  const handleSubmit = async (e) => {
    e.preventDefault();
    await createTicket(formData);
    onSubmit();
  };

  return (
    <form onSubmit={handleSubmit} className="ticket-form">
      <input
        type="text"
        placeholder="Subject"
        value={formData.subject}
        onChange={(e) => setFormData({ ...formData, subject: e.target.value })}
        required
      />
      
      <textarea
        placeholder="Describe your issue..."
        value={formData.description}
        onChange={(e) => setFormData({ ...formData, description: e.target.value })}
        required
      />
      
      <button type="button" onClick={handleAIClassification}>
        🤖 AI Classify
      </button>
      
      {aiSuggestions && (
        <div className="ai-suggestions">
          <p>AI Suggestions: Category: {aiSuggestions.category}, Priority: {aiSuggestions.priority}</p>
        </div>
      )}
      
      <select
        value={formData.category}
        onChange={(e) => setFormData({ ...formData, category: e.target.value })}
        required
      >
        <option value="">Select Category</option>
        <option value="technical">Technical</option>
        <option value="billing">Billing</option>
        <option value="general">General</option>
      </select>
      
      <select
        value={formData.priority}
        onChange={(e) => setFormData({ ...formData, priority: e.target.value })}
        required
      >
        <option value="">Select Priority</option>
        <option value="low">Low</option>
        <option value="medium">Medium</option>
        <option value="high">High</option>
      </select>
      
      <button type="submit">Submit Ticket</button>
    </form>
  );
};

export default TicketForm;
```

### Admin Analytics Dashboard

```javascript
import React, { useState, useEffect } from 'react';
import { Chart as ChartJS, ArcElement, CategoryScale, LinearScale, BarElement, Title, Tooltip, Legend } from 'chart.js';
import { Pie, Bar } from 'react-chartjs-2';

ChartJS.register(ArcElement, CategoryScale, LinearScale, BarElement, Title, Tooltip, Legend);

const AdminDashboard = () => {
  const [analytics, setAnalytics] = useState(null);
  const [risks, setRisks] = useState([]);

  useEffect(() => {
    fetchAnalytics();
    fetchRiskAlerts();
  }, []);

  const fetchAnalytics = async () => {
    const response = await authenticatedFetch('http://localhost:5000/api/analytics/dashboard');
    setAnalytics(await response.json());
  };

  const fetchRiskAlerts = async () => {
    const response = await authenticatedFetch('http://localhost:5000/api/analytics/risk-alerts');
    setRisks(await response.json());
  };

  if (!analytics) return <div>Loading...</div>;

  const taskStatusData = {
    labels: ['To Do', 'In Progress', 'Done'],
    datasets: [{
      data: [analytics.tasks.todo, analytics.tasks.inProgress, analytics.tasks.done],
      backgroundColor: ['#ff6b6b', '#ffd93d', '#6bcf7f']
    }]
  };

  const ticketCategoryData = {
    labels: analytics.tickets.categories.map(c => c.name),
    datasets: [{
      label: 'Tickets by Category',
      data: analytics.tickets.categories.map(c => c.count),
      backgroundColor: '#4ecdc4'
    }]
  };

  return (
    <div className="admin-dashboard">
      <h1>Admin Dashboard</h1>
      
      <div className="stats-grid">
        <div className="stat-card">
          <h3>Total Users</h3>
          <p>{analytics.users.total}</p>
        </div>
        <div className="stat-card">
          <h3>Active Tasks</h3>
          <p>{analytics.tasks.active}</p>
        </div>
        <div className="stat-card">
          <h3>Open Tickets</h3>
          <p>{analytics.tickets.open}</p>
        </div>
        <div className="stat-card alert">
          <h3>Risk Alerts</h3>
          <p>{risks.length}</p>
        </div>
      </div>

      <div className="charts-grid">
        <div className="chart-container">
          <h3>Task Distribution</h3>
          <Pie data={taskStatusData} />
        </div>
        <div className="chart-container">
          <h3>Ticket Categories</h3>
          <Bar data={ticketCategoryData} />
        </div>
      </div>

      <div className="risk-alerts">
        <h3>⚠️ Risk Alerts</h3>
        {risks.map(risk => (
          <div key={risk.id} className={`risk-alert risk-${risk.level}`}>
            <strong>{risk.user}</strong>: {risk.message}
            <span className="risk-score">Risk Score: {risk.score}</span>
          </div>
        ))}
      </div>
    </div>
  );
};

export default AdminDashboard;
```

## Common Patterns

### JWT Middleware (Backend)

```javascript
// middleware/auth.js
const jwt = require('jsonwebtoken');

const authMiddleware = (req, res, next) => {
  const token = req.header('Authorization')?.replace('Bearer ', '');
  
  if (!token) {
    return res.status(401).json({ message: 'No token, authorization denied' });
  }

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch (error) {
    res.status(401).json({ message: 'Token is not valid' });
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

### MongoDB User Model

```javascript
// models/User.js
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');

const userSchema = new mongoose.Schema({
  username: { type: String, required: true, unique: true },
  email: { type: String, required: true, unique: true },
  password: { type: String, required: true },
  role: { type: String, enum: ['user', 'admin'], default: 'user' },
  department: String,
  active: { type: Boolean, default: true },
  loginAttempts: { type: Number, default: 0 },
  lastLogin: Date,
  createdAt: { type: Date, default: Date.now }
});

userSchema.pre('save', async function(next) {
  if (!this.isModified('password')) return next();
  this.password = await bcrypt.hash(this.password, 10);
  next();
});

userSchema.methods.comparePassword = async function(candidatePassword) {
  return await bcrypt.compare(candidatePassword, this.password);
};

module.exports = mongoose.model('User', userSchema);
```

### Real-time Notifications with Socket.io

```javascript
// Backend: server.js
const socketIo = require('socket.io');
const io = socketIo(server, {
  cors: { origin: process.env.FRONTEND_URL }
});

io.on('connection', (socket) => {
  socket.on('join', (userId) => {
    socket.join(`user_${userId}`);
  });
});

// Emit notification
const sendNotification = (userId, notification) => {
  io.to(`user_${userId}`).emit('notification', notification);
};

module.exports = { io, sendNotification };

// Frontend: useNotifications hook
import { useEffect, useState } from 'react';
import io from 'socket.io-client';

const useNotifications = (userId) => {
  const [notifications, setNotifications] = useState([]);
  const [socket, setSocket] = useState(null);

  useEffect(() => {
    const newSocket = io('http://localhost:5000');
    setSocket(newSocket);
    
    newSocket.emit('join', userId);
    
    newSocket.on('notification', (notification) => {
      setNotifications(prev => [notification, ...prev]);
    });

    return () => newSocket.close();
  }, [userId]);

  return notifications;
};

export default useNotifications;
```

## Troubleshooting

### JWT Token Expiration Issues

```javascript
// Add token refresh logic
const refreshToken = async () => {
  try {
    const response = await fetch('http://localhost:5000/api/auth/refresh', {
      method: 'POST',
      headers: { 'Authorization': `Bearer ${localStorage.getItem('token')}` }
    });
    const data = await response.json();
    localStorage.setItem('token', data.token);
    return data.token;
  } catch (error) {
    // Token refresh failed, redirect to login
    localStorage.removeItem('token');
    window.location.href = '/login';
  }
};

// Axios interceptor for auto-refresh
axios.interceptors.response.use(
  response => response,
  async error => {
    if (error.response?.status === 401) {
      await refreshToken();
      return axios(error.config);
    }
    return Promise.reject(error);
  }
);
```

### MongoDB Connection Issues

```javascript
// backend/config/database.js
const mongoose = require('mongoose');

const connectDB = async () => {
  try {
    await mongoose.connect(process.env.MONGODB_URI, {
      useNewUrlParser: true,
      useUnifiedTopology: true,
      serverSelectionTimeoutMS: 5000
    });
    console.log('MongoDB connected successfully');
  } catch (error) {
    console.error('MongoDB connection error:', error);
    // Retry connection after 5 seconds
    setTimeout(connectDB, 5000);
  }
};

module.exports = connectDB;
```

### ML Model Training and Updates

```python
# ml-service/models/online_learner.py
from river import tree, metrics
import pickle
import os

class OnlineLearner:
    def __init__(self, model_path='models/classifier.pkl'):
        self.model_path = model_path
        if os.path.exists(model_path):
            with open(model_path, 'rb') as f:
                self.model = pickle.load(f)
        else:
            self.model = tree.HoeffdingTreeClassifier()
        
        self.metric = metrics.Accuracy()
    
    def predict(self, features):
        return self.model.predict_one(features)
    
    def update(self, features, label):
        # Online learning: update model with new data
        prediction = self.model.predict_one(features)
        self.model.learn_one(features, label)
        self.metric.update(label, prediction)
        
        # Save model periodically
        self.save_model()
    
    def save_model(self):
        os.makedirs(os.path.dirname(self.model_path), exist_ok=True)
        with open(self.model_path, 'wb') as f:
            pickle.dump(self.model, f)
```

### CORS Issues

```javascript
// backend/server.js
const cors = require('cors');

app.use(cors({
  origin: process.env.FRONTEND_URL || 'http://localhost:3000',
  credentials: true,
  methods: ['GET', 'POST', 'PUT', 'DELETE', 'PATCH'],
  allowedHeaders: ['Content-Type', 'Authorization']
}));
```

### Performance Optimization

```javascript
// Implement pagination for large datasets
const getPaginatedUsers = async (page = 1, limit = 20) => {
  const response = await authenticatedFetch(
    `http://localhost:5000/api/users?page=${page}&limit=${limit}`
  );
  return await response.json();
};

// Backend pagination
router.get('/users', authMiddleware, adminOnly, async (req, res) => {
  const page = parseInt(req.query.page) || 1;
  const limit = parseInt(req.query.limit) || 20;
  const skip = (page - 1) * limit;

  const users = await User.find()
    .select('-password')
    .skip(skip)
    .limit(limit)
    .sort({ createdAt: -1 });
  
  const total = await User.countDocuments();

  res.json({
    users,
    pagination: {
      page,
      limit,
      total,
      pages: Math.ceil(total / limit)
    }
  });
});
```

This system provides a complete enterprise-grade solution for user management with cutting-edge AI capabilities. The modular architecture allows easy extension and customization for specific organizational needs.
