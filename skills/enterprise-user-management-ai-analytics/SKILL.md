---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics, task tracking, and intelligent ticket routing
triggers:
  - "help me set up the enterprise user management system"
  - "how do I integrate AI analytics into user management"
  - "show me how to implement task tracking with Kanban boards"
  - "create a user management dashboard with AI insights"
  - "build a support ticket system with AI classification"
  - "implement risk detection and anomaly analysis for users"
  - "set up JWT authentication for enterprise user system"
  - "configure the ML service for burnout detection"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## What This Project Does

Enterprise User Management System with AI Analytics is a full-stack application that combines user management, task tracking, and support ticketing with AI-powered insights. The system features:

- **User Management**: Role-based access control, authentication, and user CRUD operations
- **Task Management**: Kanban board workflow (To Do → In Progress → Done) with time tracking
- **Support Tickets**: AI-classified ticket routing and smart prioritization
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, and project delay prediction
- **Dashboards**: Admin and user-specific analytics with real-time insights

The architecture consists of three main services:
1. **Frontend** (React.js) - User interface and dashboards
2. **Backend** (Node.js) - REST API, authentication, and business logic
3. **ML Service** (FastAPI + scikit-learn) - AI models for predictions and analytics

## Installation

### Prerequisites

- Node.js 14+ and npm
- Python 3.8+
- MongoDB (local or cloud instance)

### Clone and Setup

```bash
# Clone the repository
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

Create `.env` file in the **backend** directory:

```env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
JWT_SECRET=your_jwt_secret_key
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
```

Create `.env` file in the **ml-service** directory:

```env
MODEL_PATH=./models
LOG_LEVEL=info
BACKEND_URL=http://localhost:5000
```

### Running the Services

```bash
# Terminal 1: Start Backend
cd backend
npm start
# Runs on http://localhost:5000

# Terminal 2: Start ML Service
cd ml-service
uvicorn main:app --reload
# Runs on http://localhost:8000

# Terminal 3: Start Frontend
cd frontend
npm start
# Runs on http://localhost:3000
```

## Backend API Reference

### Authentication Endpoints

```javascript
// POST /api/auth/register
// Register new user
const registerUser = async (userData) => {
  const response = await fetch('http://localhost:5000/api/auth/register', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      username: userData.username,
      email: userData.email,
      password: userData.password,
      role: userData.role || 'user'
    })
  });
  return response.json();
};

// POST /api/auth/login
// Authenticate user and receive JWT token
const loginUser = async (credentials) => {
  const response = await fetch('http://localhost:5000/api/auth/login', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      email: credentials.email,
      password: credentials.password
    })
  });
  const data = await response.json();
  // Store token for subsequent requests
  localStorage.setItem('token', data.token);
  return data;
};
```

### User Management Endpoints

```javascript
// GET /api/users - Get all users (Admin only)
const getAllUsers = async (token) => {
  const response = await fetch('http://localhost:5000/api/users', {
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    }
  });
  return response.json();
};

// PUT /api/users/:id - Update user
const updateUser = async (userId, updates, token) => {
  const response = await fetch(`http://localhost:5000/api/users/${userId}`, {
    method: 'PUT',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify(updates)
  });
  return response.json();
};

// DELETE /api/users/:id - Delete user (Admin only)
const deleteUser = async (userId, token) => {
  const response = await fetch(`http://localhost:5000/api/users/${userId}`, {
    method: 'DELETE',
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return response.json();
};
```

### Task Management Endpoints

```javascript
// POST /api/tasks - Create new task
const createTask = async (taskData, token) => {
  const response = await fetch('http://localhost:5000/api/tasks', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      title: taskData.title,
      description: taskData.description,
      assignedTo: taskData.assignedTo,
      priority: taskData.priority, // 'low', 'medium', 'high'
      status: 'todo', // 'todo', 'inprogress', 'done'
      dueDate: taskData.dueDate
    })
  });
  return response.json();
};

// GET /api/tasks/user/:userId - Get user's tasks
const getUserTasks = async (userId, token) => {
  const response = await fetch(`http://localhost:5000/api/tasks/user/${userId}`, {
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return response.json();
};

// PATCH /api/tasks/:id/status - Update task status
const updateTaskStatus = async (taskId, newStatus, token) => {
  const response = await fetch(`http://localhost:5000/api/tasks/${taskId}/status`, {
    method: 'PATCH',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({ status: newStatus })
  });
  return response.json();
};

// POST /api/tasks/:id/time - Track time on task
const trackTaskTime = async (taskId, timeData, token) => {
  const response = await fetch(`http://localhost:5000/api/tasks/${taskId}/time`, {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      startTime: timeData.startTime,
      endTime: timeData.endTime,
      duration: timeData.duration // in seconds
    })
  });
  return response.json();
};
```

### Support Ticket Endpoints

```javascript
// POST /api/tickets - Create support ticket
const createTicket = async (ticketData, token) => {
  const response = await fetch('http://localhost:5000/api/tickets', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      title: ticketData.title,
      description: ticketData.description,
      category: ticketData.category, // Optional, AI will classify
      priority: ticketData.priority
    })
  });
  return response.json();
};

// GET /api/tickets - Get all tickets (with filters)
const getTickets = async (filters, token) => {
  const queryParams = new URLSearchParams(filters);
  const response = await fetch(`http://localhost:5000/api/tickets?${queryParams}`, {
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return response.json();
};

// PUT /api/tickets/:id - Update ticket
const updateTicket = async (ticketId, updates, token) => {
  const response = await fetch(`http://localhost:5000/api/tickets/${ticketId}`, {
    method: 'PUT',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify(updates)
  });
  return response.json();
};
```

## ML Service API Reference

### AI Analytics Endpoints

```javascript
// POST /api/ml/risk-prediction - Predict user risk level
const predictUserRisk = async (userData) => {
  const response = await fetch('http://localhost:8000/api/ml/risk-prediction', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      userId: userData.userId,
      loginFrequency: userData.loginFrequency,
      taskCompletionRate: userData.taskCompletionRate,
      averageResponseTime: userData.averageResponseTime,
      failedLoginAttempts: userData.failedLoginAttempts
    })
  });
  return response.json();
  // Returns: { riskLevel: 'low|medium|high', score: 0.75, factors: [...] }
};

// POST /api/ml/anomaly-detection - Detect anomalous behavior
const detectAnomaly = async (behaviorData) => {
  const response = await fetch('http://localhost:8000/api/ml/anomaly-detection', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      userId: behaviorData.userId,
      actions: behaviorData.actions, // Array of recent actions
      timestamp: behaviorData.timestamp
    })
  });
  return response.json();
  // Returns: { isAnomaly: true, anomalyScore: 0.85, details: '...' }
};

// POST /api/ml/burnout-detection - Analyze workload for burnout risk
const detectBurnout = async (workloadData) => {
  const response = await fetch('http://localhost:8000/api/ml/burnout-detection', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      userId: workloadData.userId,
      tasksAssigned: workloadData.tasksAssigned,
      tasksCompleted: workloadData.tasksCompleted,
      averageWorkHours: workloadData.averageWorkHours,
      weeklyOvertime: workloadData.weeklyOvertime
    })
  });
  return response.json();
  // Returns: { burnoutRisk: 'low|medium|high', recommendations: [...] }
};

// POST /api/ml/ticket-classification - Classify and route ticket
const classifyTicket = async (ticketData) => {
  const response = await fetch('http://localhost:8000/api/ml/ticket-classification', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      title: ticketData.title,
      description: ticketData.description
    })
  });
  return response.json();
  // Returns: { category: 'technical|billing|access', priority: 'high', suggestedAssignee: 'teamId' }
};

// POST /api/ml/project-insights - Predict project delays
const getProjectInsights = async (projectData) => {
  const response = await fetch('http://localhost:8000/api/ml/project-insights', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      projectId: projectData.projectId,
      totalTasks: projectData.totalTasks,
      completedTasks: projectData.completedTasks,
      daysRemaining: projectData.daysRemaining,
      teamSize: projectData.teamSize
    })
  });
  return response.json();
  // Returns: { delayProbability: 0.65, estimatedCompletion: '2024-06-15', risks: [...] }
};
```

## Frontend Integration Patterns

### React Component: User Dashboard

```javascript
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const UserDashboard = () => {
  const [tasks, setTasks] = useState([]);
  const [userStats, setUserStats] = useState({});
  const [loading, setLoading] = useState(true);
  
  const token = localStorage.getItem('token');
  const userId = localStorage.getItem('userId');

  useEffect(() => {
    fetchUserData();
  }, []);

  const fetchUserData = async () => {
    try {
      const [tasksRes, statsRes] = await Promise.all([
        axios.get(`http://localhost:5000/api/tasks/user/${userId}`, {
          headers: { Authorization: `Bearer ${token}` }
        }),
        axios.get(`http://localhost:5000/api/users/${userId}/stats`, {
          headers: { Authorization: `Bearer ${token}` }
        })
      ]);

      setTasks(tasksRes.data);
      setUserStats(statsRes.data);
      setLoading(false);
    } catch (error) {
      console.error('Error fetching user data:', error);
      setLoading(false);
    }
  };

  const moveTask = async (taskId, newStatus) => {
    try {
      await axios.patch(
        `http://localhost:5000/api/tasks/${taskId}/status`,
        { status: newStatus },
        { headers: { Authorization: `Bearer ${token}` } }
      );
      fetchUserData(); // Refresh tasks
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  if (loading) return <div>Loading...</div>;

  return (
    <div className="dashboard">
      <h1>My Dashboard</h1>
      
      <div className="stats-cards">
        <div className="stat-card">
          <h3>Tasks Completed</h3>
          <p>{userStats.completedTasks}</p>
        </div>
        <div className="stat-card">
          <h3>In Progress</h3>
          <p>{userStats.inProgressTasks}</p>
        </div>
        <div className="stat-card">
          <h3>Completion Rate</h3>
          <p>{userStats.completionRate}%</p>
        </div>
      </div>

      <div className="kanban-board">
        <div className="kanban-column">
          <h2>To Do</h2>
          {tasks.filter(t => t.status === 'todo').map(task => (
            <TaskCard 
              key={task._id} 
              task={task} 
              onMove={(status) => moveTask(task._id, status)} 
            />
          ))}
        </div>
        <div className="kanban-column">
          <h2>In Progress</h2>
          {tasks.filter(t => t.status === 'inprogress').map(task => (
            <TaskCard 
              key={task._id} 
              task={task} 
              onMove={(status) => moveTask(task._id, status)} 
            />
          ))}
        </div>
        <div className="kanban-column">
          <h2>Done</h2>
          {tasks.filter(t => t.status === 'done').map(task => (
            <TaskCard key={task._id} task={task} />
          ))}
        </div>
      </div>
    </div>
  );
};

const TaskCard = ({ task, onMove }) => (
  <div className="task-card">
    <h4>{task.title}</h4>
    <p>{task.description}</p>
    <span className={`priority ${task.priority}`}>{task.priority}</span>
    {onMove && (
      <div className="task-actions">
        {task.status === 'todo' && (
          <button onClick={() => onMove('inprogress')}>Start</button>
        )}
        {task.status === 'inprogress' && (
          <>
            <button onClick={() => onMove('todo')}>Back</button>
            <button onClick={() => onMove('done')}>Complete</button>
          </>
        )}
      </div>
    )}
  </div>
);

export default UserDashboard;
```

### React Component: Admin Analytics with AI Insights

```javascript
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const AdminAnalytics = () => {
  const [users, setUsers] = useState([]);
  const [riskAnalysis, setRiskAnalysis] = useState([]);
  const [burnoutWarnings, setBurnoutWarnings] = useState([]);
  
  const token = localStorage.getItem('token');

  useEffect(() => {
    fetchAnalytics();
  }, []);

  const fetchAnalytics = async () => {
    try {
      // Fetch all users
      const usersRes = await axios.get('http://localhost:5000/api/users', {
        headers: { Authorization: `Bearer ${token}` }
      });
      setUsers(usersRes.data);

      // Run AI analysis on each user
      const riskPromises = usersRes.data.map(async (user) => {
        const riskRes = await axios.post(
          'http://localhost:8000/api/ml/risk-prediction',
          {
            userId: user._id,
            loginFrequency: user.loginFrequency || 5,
            taskCompletionRate: user.taskCompletionRate || 0.8,
            averageResponseTime: user.avgResponseTime || 120,
            failedLoginAttempts: user.failedLogins || 0
          }
        );
        return { userId: user._id, username: user.username, ...riskRes.data };
      });

      const burnoutPromises = usersRes.data.map(async (user) => {
        const burnoutRes = await axios.post(
          'http://localhost:8000/api/ml/burnout-detection',
          {
            userId: user._id,
            tasksAssigned: user.tasksAssigned || 10,
            tasksCompleted: user.tasksCompleted || 7,
            averageWorkHours: user.avgWorkHours || 8,
            weeklyOvertime: user.weeklyOvertime || 0
          }
        );
        return { userId: user._id, username: user.username, ...burnoutRes.data };
      });

      const risks = await Promise.all(riskPromises);
      const burnouts = await Promise.all(burnoutPromises);

      setRiskAnalysis(risks.filter(r => r.riskLevel !== 'low'));
      setBurnoutWarnings(burnouts.filter(b => b.burnoutRisk !== 'low'));
    } catch (error) {
      console.error('Error fetching analytics:', error);
    }
  };

  return (
    <div className="admin-analytics">
      <h1>Admin Analytics Dashboard</h1>

      <section className="alerts-section">
        <h2>Risk Alerts</h2>
        {riskAnalysis.length > 0 ? (
          <div className="alert-grid">
            {riskAnalysis.map((risk) => (
              <div key={risk.userId} className={`alert-card ${risk.riskLevel}`}>
                <h3>{risk.username}</h3>
                <p>Risk Level: <strong>{risk.riskLevel}</strong></p>
                <p>Score: {(risk.score * 100).toFixed(0)}%</p>
                <ul>
                  {risk.factors?.map((factor, idx) => (
                    <li key={idx}>{factor}</li>
                  ))}
                </ul>
              </div>
            ))}
          </div>
        ) : (
          <p>No high-risk users detected</p>
        )}
      </section>

      <section className="burnout-section">
        <h2>Burnout Warnings</h2>
        {burnoutWarnings.length > 0 ? (
          <div className="alert-grid">
            {burnoutWarnings.map((burnout) => (
              <div key={burnout.userId} className={`alert-card ${burnout.burnoutRisk}`}>
                <h3>{burnout.username}</h3>
                <p>Burnout Risk: <strong>{burnout.burnoutRisk}</strong></p>
                <div className="recommendations">
                  <h4>Recommendations:</h4>
                  <ul>
                    {burnout.recommendations?.map((rec, idx) => (
                      <li key={idx}>{rec}</li>
                    ))}
                  </ul>
                </div>
              </div>
            ))}
          </div>
        ) : (
          <p>No burnout risks detected</p>
        )}
      </section>

      <section className="users-overview">
        <h2>Users Overview</h2>
        <table>
          <thead>
            <tr>
              <th>Username</th>
              <th>Email</th>
              <th>Role</th>
              <th>Tasks</th>
              <th>Status</th>
            </tr>
          </thead>
          <tbody>
            {users.map((user) => (
              <tr key={user._id}>
                <td>{user.username}</td>
                <td>{user.email}</td>
                <td>{user.role}</td>
                <td>{user.tasksAssigned || 0} / {user.tasksCompleted || 0}</td>
                <td>{user.status || 'active'}</td>
              </tr>
            ))}
          </tbody>
        </table>
      </section>
    </div>
  );
};

export default AdminAnalytics;
```

### Time Tracking Component

```javascript
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const TaskTimer = ({ taskId }) => {
  const [isRunning, setIsRunning] = useState(false);
  const [elapsedTime, setElapsedTime] = useState(0);
  const [startTime, setStartTime] = useState(null);
  
  const token = localStorage.getItem('token');

  useEffect(() => {
    let interval;
    if (isRunning) {
      interval = setInterval(() => {
        setElapsedTime(Math.floor((Date.now() - startTime) / 1000));
      }, 1000);
    }
    return () => clearInterval(interval);
  }, [isRunning, startTime]);

  const startTimer = () => {
    setStartTime(Date.now());
    setIsRunning(true);
  };

  const stopTimer = async () => {
    setIsRunning(false);
    const endTime = Date.now();
    
    try {
      await axios.post(
        `http://localhost:5000/api/tasks/${taskId}/time`,
        {
          startTime: new Date(startTime),
          endTime: new Date(endTime),
          duration: elapsedTime
        },
        { headers: { Authorization: `Bearer ${token}` } }
      );
      
      // Reset timer
      setElapsedTime(0);
      setStartTime(null);
    } catch (error) {
      console.error('Error saving time entry:', error);
    }
  };

  const formatTime = (seconds) => {
    const hrs = Math.floor(seconds / 3600);
    const mins = Math.floor((seconds % 3600) / 60);
    const secs = seconds % 60;
    return `${hrs.toString().padStart(2, '0')}:${mins.toString().padStart(2, '0')}:${secs.toString().padStart(2, '0')}`;
  };

  return (
    <div className="task-timer">
      <div className="timer-display">{formatTime(elapsedTime)}</div>
      <div className="timer-controls">
        {!isRunning ? (
          <button onClick={startTimer} className="btn-start">Start</button>
        ) : (
          <button onClick={stopTimer} className="btn-stop">Stop</button>
        )}
      </div>
    </div>
  );
};

export default TaskTimer;
```

### Support Ticket Creation with AI Classification

```javascript
import React, { useState } from 'react';
import axios from 'axios';

const CreateTicket = () => {
  const [formData, setFormData] = useState({
    title: '',
    description: '',
    priority: 'medium'
  });
  const [aiSuggestions, setAiSuggestions] = useState(null);
  const [submitting, setSubmitting] = useState(false);
  
  const token = localStorage.getItem('token');

  const handleInputChange = (e) => {
    setFormData({ ...formData, [e.target.name]: e.target.value });
  };

  const classifyTicket = async () => {
    if (!formData.title || !formData.description) return;

    try {
      const response = await axios.post(
        'http://localhost:8000/api/ml/ticket-classification',
        {
          title: formData.title,
          description: formData.description
        }
      );
      setAiSuggestions(response.data);
    } catch (error) {
      console.error('Error classifying ticket:', error);
    }
  };

  const handleSubmit = async (e) => {
    e.preventDefault();
    setSubmitting(true);

    try {
      const ticketData = {
        ...formData,
        category: aiSuggestions?.category || 'general',
        aiClassified: !!aiSuggestions
      };

      await axios.post(
        'http://localhost:5000/api/tickets',
        ticketData,
        { headers: { Authorization: `Bearer ${token}` } }
      );

      alert('Ticket created successfully!');
      setFormData({ title: '', description: '', priority: 'medium' });
      setAiSuggestions(null);
    } catch (error) {
      console.error('Error creating ticket:', error);
      alert('Failed to create ticket');
    } finally {
      setSubmitting(false);
    }
  };

  return (
    <div className="create-ticket">
      <h2>Create Support Ticket</h2>
      <form onSubmit={handleSubmit}>
        <div className="form-group">
          <label>Title</label>
          <input
            type="text"
            name="title"
            value={formData.title}
            onChange={handleInputChange}
            onBlur={classifyTicket}
            required
          />
        </div>

        <div className="form-group">
          <label>Description</label>
          <textarea
            name="description"
            value={formData.description}
            onChange={handleInputChange}
            onBlur={classifyTicket}
            rows="5"
            required
          />
        </div>

        <div className="form-group">
          <label>Priority</label>
          <select
            name="priority"
            value={formData.priority}
            onChange={handleInputChange}
          >
            <option value="low">Low</option>
            <option value="medium">Medium</option>
            <option value="high">High</option>
          </select>
        </div>

        {aiSuggestions && (
          <div className="ai-suggestions">
            <h3>AI Analysis</h3>
            <p><strong>Suggested Category:</strong> {aiSuggestions.category}</p>
            <p><strong>Recommended Priority:</strong> {aiSuggestions.priority}</p>
            <p><strong>Suggested Assignee:</strong> {aiSuggestions.suggestedAssignee}</p>
          </div>
        )}

        <button type="submit" disabled={submitting}>
          {submitting ? 'Creating...' : 'Create Ticket'}
        </button>
      </form>
    </div>
  );
};

export default CreateTicket;
```

## Configuration

### Backend Configuration (backend/config.js)

```javascript
module.exports = {
  server: {
    port: process.env.PORT || 5000,
    env: process.env.NODE_ENV || 'development'
  },
  database: {
    uri: process.env.MONGODB_URI || 'mongodb://localhost:27017/enterprise_user_mgmt',
    options: {
      useNewUrlParser: true,
      useUnifiedTopology: true
    }
  },
  jwt: {
    secret: process.env.JWT_SECRET,
    expiresIn: process.env.JWT_EXPIRE || '7d'
  },
  mlService: {
    url: process.env.ML_SERVICE_URL || 'http://localhost:8000'
  },
  cors: {
    origin: process.env.FRONTEND_URL || 'http://localhost:3000',
    credentials: true
  }
};
```

### ML Service Configuration (ml-service/config.py)

```python
import os
from pathlib import Path

class Config:
    # Model paths
    MODEL_PATH = os.getenv('MODEL_PATH', './models')
    RISK_MODEL = Path(MODEL_PATH) / 'risk_model.pkl'
    ANOMALY_MODEL = Path(MODEL_PATH) / 'anomaly_model.pkl'
    BURNOUT_MODEL = Path(MODEL_PATH) / 'burnout_model.pkl'
    TICKET_CLASSIFIER = Path(MODEL_PATH) / 'ticket_classifier.pkl'
    
    # Logging
    LOG_LEVEL = os.getenv('LOG_LEVEL', 'INFO')
    
    # Backend integration
    BACKEND_URL = os.getenv('BACKEND_URL', 'http://localhost:5000')
    
    # Model parameters
    RISK_THRESHOLD = 0.7
    ANOMALY_THRESHOLD = 0.8
    BURNOUT_THRESHOLD = 0.6
    
    # Update frequency (in hours)
    MODEL_UPDATE_FREQUENCY = 24
```

## Common Patterns

### Authenticated API Request Helper

```javascript
// frontend/src/utils/api.js
import axios from 'axios';

const API_BASE = process.env.REACT_APP_API_URL || 'http://localhost:5000/api';

const apiClient = axios.create({
  baseURL: API_BASE,
  timeout: 10000
});

// Add token to all requests
apiClient.interceptors.request.use(
  (config) => {
    const token
