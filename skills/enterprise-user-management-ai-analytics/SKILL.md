---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics, risk detection, and task management
triggers:
  - "set up enterprise user management system"
  - "create user management dashboard with AI analytics"
  - "implement AI-powered task tracking system"
  - "build user management with anomaly detection"
  - "configure AI risk prediction for users"
  - "add AI ticket classification system"
  - "create admin dashboard with AI insights"
  - "implement burnout detection for employees"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection

## Overview

Enterprise User Management System is a full-stack JavaScript application that combines traditional user and task management with AI-powered analytics. It provides:

- **User Management**: Role-based access control, authentication via JWT
- **Task Management**: Kanban boards, time tracking, task assignment
- **Support System**: Ticket management with AI classification
- **AI Analytics**: Risk detection, anomaly detection, burnout analysis, predictive insights
- **Admin Tools**: Audit logs, organizational analytics, alert systems

The system consists of three main components:
1. **Frontend** (React.js) - User interface and dashboards
2. **Backend** (Node.js) - REST APIs and business logic
3. **ML Service** (FastAPI + Python) - AI/ML features

## Installation

### Prerequisites

```bash
# Required
node >= 14.x
npm >= 6.x
python >= 3.8
mongodb >= 4.x
```

### Full Stack Setup

```bash
# Clone repository
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics

# Install backend dependencies
cd backend
npm install

# Install ML service dependencies
cd ../ml-service
pip install -r requirements.txt

# Install frontend dependencies
cd ../frontend
npm install
```

### Environment Configuration

**Backend (.env in /backend)**
```bash
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
JWT_SECRET=your_jwt_secret_key
JWT_EXPIRES_IN=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
```

**ML Service (.env in /ml-service)**
```bash
PORT=8000
MODEL_PATH=./models
LOG_LEVEL=INFO
```

**Frontend (.env in /frontend)**
```bash
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
```

## Running the Application

### Start All Services

```bash
# Terminal 1 - Backend
cd backend
npm start
# Runs at http://localhost:5000

# Terminal 2 - ML Service
cd ml-service
uvicorn main:app --reload
# Runs at http://localhost:8000

# Terminal 3 - Frontend
cd frontend
npm start
# Runs at http://localhost:3000
```

### Development Mode

```bash
# Backend with nodemon
cd backend
npm run dev

# Frontend with hot reload (already enabled)
cd frontend
npm start
```

## Backend API Reference

### Authentication APIs

```javascript
// User Registration
POST /api/auth/register
Content-Type: application/json

{
  "username": "john.doe",
  "email": "john@example.com",
  "password": "SecurePass123",
  "role": "user" // or "admin"
}

// User Login
POST /api/auth/login
{
  "email": "john@example.com",
  "password": "SecurePass123"
}

// Response includes JWT token
{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "user": {
    "id": "user_id",
    "username": "john.doe",
    "role": "user"
  }
}
```

### User Management APIs

```javascript
// Get All Users (Admin only)
GET /api/users
Headers: { "Authorization": "Bearer <token>" }

// Get User by ID
GET /api/users/:id
Headers: { "Authorization": "Bearer <token>" }

// Update User
PUT /api/users/:id
Headers: { "Authorization": "Bearer <token>" }
{
  "username": "jane.doe",
  "email": "jane@example.com",
  "role": "admin"
}

// Delete User (Admin only)
DELETE /api/users/:id
Headers: { "Authorization": "Bearer <token>" }
```

### Task Management APIs

```javascript
// Create Task
POST /api/tasks
Headers: { "Authorization": "Bearer <token>" }
{
  "title": "Implement AI feature",
  "description": "Add anomaly detection",
  "assignedTo": "user_id",
  "status": "todo", // todo, in-progress, done
  "priority": "high",
  "dueDate": "2026-05-01"
}

// Get Tasks (filtered by user)
GET /api/tasks?userId=<user_id>&status=todo

// Update Task Status
PATCH /api/tasks/:id/status
{
  "status": "in-progress"
}

// Track Time on Task
POST /api/tasks/:id/time
{
  "timeSpent": 3600 // seconds
}
```

### Support Ticket APIs

```javascript
// Create Support Ticket
POST /api/tickets
Headers: { "Authorization": "Bearer <token>" }
{
  "subject": "Cannot access dashboard",
  "description": "Getting 403 error when loading dashboard",
  "priority": "high",
  "category": "technical"
}

// Get Tickets
GET /api/tickets?status=open&userId=<user_id>

// Update Ticket
PATCH /api/tickets/:id
{
  "status": "in-progress",
  "assignedTo": "admin_id"
}
```

## ML Service API Reference

### AI Analytics Endpoints

```javascript
// Risk Prediction
POST /api/ml/predict-risk
Content-Type: application/json

{
  "userId": "user_id",
  "features": {
    "taskCompletionRate": 0.75,
    "averageResponseTime": 120,
    "missedDeadlines": 3,
    "ticketCount": 5,
    "loginFrequency": 0.9
  }
}

// Response
{
  "riskScore": 0.65,
  "riskLevel": "medium", // low, medium, high
  "factors": ["missedDeadlines", "ticketCount"],
  "recommendations": ["Review workload", "Provide support"]
}
```

```javascript
// Anomaly Detection
POST /api/ml/detect-anomaly
{
  "userId": "user_id",
  "behavior": {
    "loginTime": "03:00 AM",
    "loginLocation": "Unknown IP",
    "accessPattern": "unusual",
    "dataAccess": ["sensitive_data"]
  }
}

// Response
{
  "isAnomaly": true,
  "anomalyScore": 0.85,
  "type": "security",
  "alert": "Unusual login time and location detected"
}
```

```javascript
// Burnout Detection
POST /api/ml/detect-burnout
{
  "userId": "user_id",
  "metrics": {
    "hoursWorked": 65,
    "tasksCompleted": 25,
    "stressLevel": 8,
    "overtimeHours": 15,
    "consecutiveDays": 12
  }
}

// Response
{
  "burnoutRisk": "high",
  "score": 0.78,
  "indicators": ["excessive_hours", "no_breaks"],
  "recommendation": "Immediate workload reduction needed"
}
```

```javascript
// Ticket Classification
POST /api/ml/classify-ticket
{
  "subject": "Cannot access dashboard",
  "description": "Getting 403 error when loading dashboard",
  "metadata": {
    "userId": "user_id",
    "timestamp": "2026-04-15T10:30:00Z"
  }
}

// Response
{
  "category": "technical",
  "priority": "high",
  "suggestedAssignee": "tech_support_team",
  "estimatedResolutionTime": "2 hours",
  "confidence": 0.92
}
```

```javascript
// Project Delay Prediction
POST /api/ml/predict-delay
{
  "projectId": "project_id",
  "features": {
    "completedTasks": 45,
    "totalTasks": 100,
    "teamSize": 8,
    "daysElapsed": 30,
    "daysRemaining": 20,
    "avgTaskCompletionTime": 2.5,
    "blockedTasks": 5
  }
}

// Response
{
  "delayProbability": 0.73,
  "estimatedDelay": 15, // days
  "riskFactors": ["blockedTasks", "completion_rate"],
  "recommendations": ["Resolve blockers", "Add resources"]
}
```

## Frontend Integration

### Authentication Setup

```javascript
// src/services/authService.js
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL;

export const authService = {
  async login(email, password) {
    const response = await axios.post(`${API_URL}/api/auth/login`, {
      email,
      password
    });
    
    if (response.data.token) {
      localStorage.setItem('token', response.data.token);
      localStorage.setItem('user', JSON.stringify(response.data.user));
    }
    
    return response.data;
  },

  logout() {
    localStorage.removeItem('token');
    localStorage.removeItem('user');
  },

  getCurrentUser() {
    return JSON.parse(localStorage.getItem('user'));
  },

  getToken() {
    return localStorage.getItem('token');
  }
};
```

### API Client with JWT

```javascript
// src/services/apiClient.js
import axios from 'axios';
import { authService } from './authService';

const apiClient = axios.create({
  baseURL: process.env.REACT_APP_API_URL
});

// Add JWT token to all requests
apiClient.interceptors.request.use(
  (config) => {
    const token = authService.getToken();
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
  },
  (error) => Promise.reject(error)
);

// Handle 401 errors
apiClient.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 401) {
      authService.logout();
      window.location.href = '/login';
    }
    return Promise.reject(error);
  }
);

export default apiClient;
```

### Task Management Component

```javascript
// src/components/TaskBoard.jsx
import React, { useState, useEffect } from 'react';
import apiClient from '../services/apiClient';

const TaskBoard = () => {
  const [tasks, setTasks] = useState({
    todo: [],
    inProgress: [],
    done: []
  });

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    try {
      const response = await apiClient.get('/api/tasks');
      const grouped = groupTasksByStatus(response.data);
      setTasks(grouped);
    } catch (error) {
      console.error('Error fetching tasks:', error);
    }
  };

  const groupTasksByStatus = (taskList) => {
    return taskList.reduce((acc, task) => {
      const status = task.status.replace('-', '');
      if (status === 'inProgress') {
        acc.inProgress.push(task);
      } else {
        acc[status].push(task);
      }
      return acc;
    }, { todo: [], inProgress: [], done: [] });
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    try {
      await apiClient.patch(`/api/tasks/${taskId}/status`, {
        status: newStatus
      });
      fetchTasks();
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  const handleDragStart = (e, taskId) => {
    e.dataTransfer.setData('taskId', taskId);
  };

  const handleDrop = async (e, newStatus) => {
    e.preventDefault();
    const taskId = e.dataTransfer.getData('taskId');
    await updateTaskStatus(taskId, newStatus);
  };

  return (
    <div className="task-board">
      {['todo', 'in-progress', 'done'].map((status) => (
        <div
          key={status}
          className="task-column"
          onDrop={(e) => handleDrop(e, status)}
          onDragOver={(e) => e.preventDefault()}
        >
          <h3>{status.toUpperCase()}</h3>
          {tasks[status.replace('-', '')].map((task) => (
            <div
              key={task.id}
              className="task-card"
              draggable
              onDragStart={(e) => handleDragStart(e, task.id)}
            >
              <h4>{task.title}</h4>
              <p>{task.description}</p>
              <span className={`priority-${task.priority}`}>
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

### AI Risk Dashboard

```javascript
// src/components/RiskDashboard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import apiClient from '../services/apiClient';

const ML_API_URL = process.env.REACT_APP_ML_API_URL;

const RiskDashboard = ({ userId }) => {
  const [riskData, setRiskData] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchRiskAnalysis();
  }, [userId]);

  const fetchRiskAnalysis = async () => {
    try {
      // Get user metrics from backend
      const metricsResponse = await apiClient.get(
        `/api/users/${userId}/metrics`
      );

      // Get AI risk prediction from ML service
      const riskResponse = await axios.post(
        `${ML_API_URL}/api/ml/predict-risk`,
        {
          userId: userId,
          features: metricsResponse.data
        }
      );

      setRiskData(riskResponse.data);
      setLoading(false);
    } catch (error) {
      console.error('Error fetching risk analysis:', error);
      setLoading(false);
    }
  };

  if (loading) return <div>Loading risk analysis...</div>;

  return (
    <div className="risk-dashboard">
      <h2>User Risk Analysis</h2>
      <div className={`risk-score risk-${riskData.riskLevel}`}>
        <span className="score">{(riskData.riskScore * 100).toFixed(0)}%</span>
        <span className="level">{riskData.riskLevel.toUpperCase()}</span>
      </div>

      <div className="risk-factors">
        <h3>Risk Factors</h3>
        <ul>
          {riskData.factors.map((factor, idx) => (
            <li key={idx}>{factor}</li>
          ))}
        </ul>
      </div>

      <div className="recommendations">
        <h3>Recommendations</h3>
        <ul>
          {riskData.recommendations.map((rec, idx) => (
            <li key={idx}>{rec}</li>
          ))}
        </ul>
      </div>
    </div>
  );
};

export default RiskDashboard;
```

### Ticket Creation with AI Classification

```javascript
// src/components/CreateTicket.jsx
import React, { useState } from 'react';
import axios from 'axios';
import apiClient from '../services/apiClient';

const ML_API_URL = process.env.REACT_APP_ML_API_URL;

const CreateTicket = () => {
  const [formData, setFormData] = useState({
    subject: '',
    description: ''
  });
  const [aiSuggestion, setAiSuggestion] = useState(null);
  const [loading, setLoading] = useState(false);

  const handleChange = (e) => {
    setFormData({
      ...formData,
      [e.target.name]: e.target.value
    });
  };

  const classifyTicket = async () => {
    if (!formData.subject || !formData.description) return;

    setLoading(true);
    try {
      const response = await axios.post(
        `${ML_API_URL}/api/ml/classify-ticket`,
        {
          subject: formData.subject,
          description: formData.description,
          metadata: {
            timestamp: new Date().toISOString()
          }
        }
      );
      setAiSuggestion(response.data);
    } catch (error) {
      console.error('Error classifying ticket:', error);
    }
    setLoading(false);
  };

  const handleSubmit = async (e) => {
    e.preventDefault();

    try {
      await apiClient.post('/api/tickets', {
        ...formData,
        category: aiSuggestion?.category || 'general',
        priority: aiSuggestion?.priority || 'medium'
      });

      alert('Ticket created successfully!');
      setFormData({ subject: '', description: '' });
      setAiSuggestion(null);
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
          name="subject"
          placeholder="Subject"
          value={formData.subject}
          onChange={handleChange}
          onBlur={classifyTicket}
          required
        />

        <textarea
          name="description"
          placeholder="Description"
          value={formData.description}
          onChange={handleChange}
          onBlur={classifyTicket}
          required
        />

        {loading && <div>AI is analyzing your ticket...</div>}

        {aiSuggestion && (
          <div className="ai-suggestion">
            <h3>AI Suggestions</h3>
            <p><strong>Category:</strong> {aiSuggestion.category}</p>
            <p><strong>Priority:</strong> {aiSuggestion.priority}</p>
            <p><strong>Estimated Resolution:</strong> {aiSuggestion.estimatedResolutionTime}</p>
            <p><strong>Confidence:</strong> {(aiSuggestion.confidence * 100).toFixed(0)}%</p>
          </div>
        )}

        <button type="submit">Create Ticket</button>
      </form>
    </div>
  );
};

export default CreateTicket;
```

## Common Patterns

### Protected Routes

```javascript
// src/components/ProtectedRoute.jsx
import React from 'react';
import { Navigate } from 'react-router-dom';
import { authService } from '../services/authService';

const ProtectedRoute = ({ children, adminOnly = false }) => {
  const user = authService.getCurrentUser();

  if (!user) {
    return <Navigate to="/login" />;
  }

  if (adminOnly && user.role !== 'admin') {
    return <Navigate to="/dashboard" />;
  }

  return children;
};

export default ProtectedRoute;

// Usage in App.js
import { BrowserRouter, Routes, Route } from 'react-router-dom';
import ProtectedRoute from './components/ProtectedRoute';
import AdminDashboard from './pages/AdminDashboard';
import UserDashboard from './pages/UserDashboard';

function App() {
  return (
    <BrowserRouter>
      <Routes>
        <Route path="/login" element={<Login />} />
        <Route
          path="/dashboard"
          element={
            <ProtectedRoute>
              <UserDashboard />
            </ProtectedRoute>
          }
        />
        <Route
          path="/admin"
          element={
            <ProtectedRoute adminOnly>
              <AdminDashboard />
            </ProtectedRoute>
          }
        />
      </Routes>
    </BrowserRouter>
  );
}
```

### Real-time Analytics Updates

```javascript
// src/hooks/useRealTimeAnalytics.js
import { useState, useEffect } from 'react';
import apiClient from '../services/apiClient';

export const useRealTimeAnalytics = (userId, interval = 30000) => {
  const [analytics, setAnalytics] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    const fetchAnalytics = async () => {
      try {
        const response = await apiClient.get(`/api/analytics/user/${userId}`);
        setAnalytics(response.data);
        setLoading(false);
      } catch (err) {
        setError(err);
        setLoading(false);
      }
    };

    fetchAnalytics();
    const intervalId = setInterval(fetchAnalytics, interval);

    return () => clearInterval(intervalId);
  }, [userId, interval]);

  return { analytics, loading, error };
};

// Usage
const UserAnalytics = ({ userId }) => {
  const { analytics, loading, error } = useRealTimeAnalytics(userId);

  if (loading) return <div>Loading...</div>;
  if (error) return <div>Error loading analytics</div>;

  return (
    <div>
      <h3>Performance Metrics</h3>
      <p>Tasks Completed: {analytics.tasksCompleted}</p>
      <p>Average Response Time: {analytics.avgResponseTime}s</p>
      <p>Completion Rate: {(analytics.completionRate * 100).toFixed(1)}%</p>
    </div>
  );
};
```

### Batch AI Processing

```javascript
// Backend utility for batch AI processing
// backend/utils/batchAIProcessor.js
const axios = require('axios');

class BatchAIProcessor {
  constructor(mlServiceUrl) {
    this.mlServiceUrl = mlServiceUrl;
  }

  async processBatchRiskAnalysis(users) {
    const promises = users.map(async (user) => {
      try {
        const response = await axios.post(
          `${this.mlServiceUrl}/api/ml/predict-risk`,
          {
            userId: user._id,
            features: this.extractUserFeatures(user)
          }
        );
        return {
          userId: user._id,
          ...response.data
        };
      } catch (error) {
        console.error(`Risk analysis failed for user ${user._id}:`, error);
        return null;
      }
    });

    const results = await Promise.all(promises);
    return results.filter(result => result !== null);
  }

  extractUserFeatures(user) {
    return {
      taskCompletionRate: user.metrics?.taskCompletionRate || 0,
      averageResponseTime: user.metrics?.averageResponseTime || 0,
      missedDeadlines: user.metrics?.missedDeadlines || 0,
      ticketCount: user.metrics?.ticketCount || 0,
      loginFrequency: user.metrics?.loginFrequency || 0
    };
  }

  async detectBurnoutForTeam(teamMembers) {
    const burnoutResults = [];

    for (const member of teamMembers) {
      const metrics = await this.getWorkloadMetrics(member._id);
      
      const response = await axios.post(
        `${this.mlServiceUrl}/api/ml/detect-burnout`,
        {
          userId: member._id,
          metrics: metrics
        }
      );

      if (response.data.burnoutRisk === 'high') {
        burnoutResults.push({
          userId: member._id,
          username: member.username,
          ...response.data
        });
      }
    }

    return burnoutResults;
  }

  async getWorkloadMetrics(userId) {
    // Fetch from your database
    // This is a placeholder implementation
    return {
      hoursWorked: 45,
      tasksCompleted: 20,
      stressLevel: 6,
      overtimeHours: 5,
      consecutiveDays: 10
    };
  }
}

module.exports = BatchAIProcessor;

// Usage in backend route
const BatchAIProcessor = require('../utils/batchAIProcessor');

router.get('/api/admin/batch-risk-analysis', async (req, res) => {
  try {
    const users = await User.find({ role: 'user' });
    const processor = new BatchAIProcessor(process.env.ML_SERVICE_URL);
    const riskAnalyses = await processor.processBatchRiskAnalysis(users);
    
    res.json(riskAnalyses);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});
```

## Database Schema Examples

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
  metrics: {
    taskCompletionRate: { type: Number, default: 0 },
    averageResponseTime: { type: Number, default: 0 },
    missedDeadlines: { type: Number, default: 0 },
    ticketCount: { type: Number, default: 0 },
    loginFrequency: { type: Number, default: 0 }
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
  this.password = await bcrypt.hash(this.password, 10);
  next();
});

// Compare password method
userSchema.methods.comparePassword = async function(candidatePassword) {
  return await bcrypt.compare(candidatePassword, this.password);
};

module.exports = mongoose.model('User', userSchema);
```

### Task Model

```javascript
// backend/models/Task.js
const mongoose = require('mongoose');

const taskSchema = new mongoose.Schema({
  title: {
    type: String,
    required: true
  },
  description: String,
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
  assignedTo: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User',
    required: true
  },
  createdBy: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User'
  },
  dueDate: Date,
  timeSpent: {
    type: Number,
    default: 0 // in seconds
  },
  createdAt: {
    type: Date,
    default: Date.now
  },
  updatedAt: Date
});

taskSchema.pre('save', function(next) {
  this.updatedAt = Date.now();
  next();
});

module.exports = mongoose.model('Task', taskSchema);
```

### Ticket Model

```javascript
// backend/models/Ticket.js
const mongoose = require('mongoose');

const ticketSchema = new mongoose.Schema({
  subject: {
    type: String,
    required: true
  },
  description: {
    type: String,
    required: true
  },
  status: {
    type: String,
    enum: ['open', 'in-progress', 'resolved', 'closed'],
    default: 'open'
  },
  priority: {
    type: String,
    enum: ['low', 'medium', 'high', 'urgent'],
    default: 'medium'
  },
  category: {
    type: String,
    enum: ['technical', 'billing', 'general', 'feature-request'],
    default: 'general'
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
  aiClassification: {
    confidence: Number,
    suggestedCategory: String,
    suggestedPriority: String,
    estimatedResolutionTime: String
  },
  createdAt: {
    type: Date,
    default: Date.now
  },
  resolvedAt: Date
});

module.exports = mongoose.model('Ticket', ticketSchema);
```

## Troubleshooting

### JWT Authentication Issues

```javascript
// Check if token is valid
const jwt = require('jsonwebtoken');

function verifyToken(token) {
  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    console.log('Token valid:', decoded);
    return decoded;
  } catch (error) {
    console.error('Token verification failed:', error.message);
    return null;
  }
}

// Middleware to refresh token
const refreshTokenMiddleware = (req, res, next) => {
  const token = req.headers.authorization?.split(' ')[1];
  
  if (!token) return next();

  const decoded = jwt.decode(token);
  const currentTime = Date.now() / 1000;

  // Refresh if token expires in less than 1 day
  if (decoded.exp - currentTime < 86400) {
    const newToken = jwt.sign(
      { id: decoded.id, role: decoded.role },
      process.env.JWT_SECRET,
      { expiresIn: process.env.JWT_EXPIRES_IN }
    );
    res.setHeader('X-New-Token', newToken);
  }

  next();
};
```

### MongoDB Connection Issues

```javascript
// backend/config/database.js
const mongoose = require('mongoose');

const connectDB = async () => {
  try {
    await mongoose.connect(process.env
