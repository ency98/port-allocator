# Port Allocator API & Database Integration Cheatsheet

## Data Models

### Core Entity Types
| Entity Type | Description |
|-------------|-------------|
| `host`      | Physical or virtual server that can host containers/services |
| `cluster`   | Group of hosts operating as a single unit (e.g., Docker Swarm, Kubernetes) |
| `container` | Individual Docker container running on a host |
| `service`   | Docker service (can be standalone or part of a stack) |
| `stack`     | Collection of related services defined in a single compose file |

### Entity Relationships
- `host` ← contains → `container`
- `host` ← belongs to → `cluster` (optional)
- `cluster` ← contains → `stack`
- `stack` ← contains → `service`
- `host`/`cluster` ← contains → `service` (standalone services)

## Database Schema

### Host
```sql
CREATE TABLE hosts (
  id UUID PRIMARY KEY,
  name VARCHAR(255) NOT NULL UNIQUE,
  location VARCHAR(255),
  fqdn VARCHAR(255),
  ip VARCHAR(45),
  owner VARCHAR(255),
  is_cluster_node BOOLEAN DEFAULT FALSE,
  cluster_id UUID,
  cluster_role VARCHAR(50),
  description TEXT,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (cluster_id) REFERENCES clusters(id)
);
```

### Cluster
```sql
CREATE TABLE clusters (
  id UUID PRIMARY KEY,
  name VARCHAR(255) NOT NULL UNIQUE,
  location VARCHAR(255),
  owner VARCHAR(255),
  has_vip BOOLEAN DEFAULT FALSE,
  vip_address VARCHAR(45),
  vip_fqdn VARCHAR(255),
  description TEXT,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### Stack
```sql
CREATE TABLE stacks (
  id UUID PRIMARY KEY,
  name VARCHAR(255) NOT NULL,
  host_id UUID,
  cluster_id UUID,
  description TEXT,
  link VARCHAR(255),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (host_id) REFERENCES hosts(id),
  FOREIGN KEY (cluster_id) REFERENCES clusters(id),
  UNIQUE(name, host_id),
  UNIQUE(name, cluster_id),
  CHECK ((host_id IS NULL AND cluster_id IS NOT NULL) OR (host_id IS NOT NULL AND cluster_id IS NULL))
);
```

### Service
```sql
CREATE TABLE services (
  id UUID PRIMARY KEY,
  name VARCHAR(255) NOT NULL,
  stack_id UUID,
  host_id UUID,
  cluster_id UUID,
  image VARCHAR(255),
  port INTEGER,
  container_port INTEGER,
  link VARCHAR(255),
  description TEXT,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (stack_id) REFERENCES stacks(id),
  FOREIGN KEY (host_id) REFERENCES hosts(id),
  FOREIGN KEY (cluster_id) REFERENCES clusters(id),
  UNIQUE(name, stack_id),
  UNIQUE(name, host_id, stack_id IS NULL),
  UNIQUE(name, cluster_id, stack_id IS NULL),
  CHECK (
    (stack_id IS NOT NULL AND host_id IS NULL AND cluster_id IS NULL) OR
    (stack_id IS NULL AND host_id IS NOT NULL AND cluster_id IS NULL) OR
    (stack_id IS NULL AND host_id IS NULL AND cluster_id IS NOT NULL)
  )
);
```

### Container
```sql
CREATE TABLE containers (
  id UUID PRIMARY KEY,
  name VARCHAR(255) NOT NULL,
  host_id UUID NOT NULL,
  image VARCHAR(255),
  port INTEGER,
  container_port INTEGER,
  exposure VARCHAR(50) DEFAULT 'internal',
  lan_dns VARCHAR(255),
  wan_dns VARCHAR(255),
  proxy_url VARCHAR(255),
  link VARCHAR(255),
  owner VARCHAR(255),
  docker_options TEXT,
  description TEXT,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (host_id) REFERENCES hosts(id),
  UNIQUE(name, host_id)
);
```

### Environment Variables
```sql
CREATE TABLE environment_variables (
  id UUID PRIMARY KEY,
  container_id UUID,
  service_id UUID,
  key VARCHAR(255) NOT NULL,
  value TEXT,
  FOREIGN KEY (container_id) REFERENCES containers(id) ON DELETE CASCADE,
  FOREIGN KEY (service_id) REFERENCES services(id) ON DELETE CASCADE,
  CHECK (
    (container_id IS NOT NULL AND service_id IS NULL) OR
    (container_id IS NULL AND service_id IS NOT NULL)
  )
);
```

### Networks
```sql
CREATE TABLE networks (
  id UUID PRIMARY KEY,
  container_id UUID,
  service_id UUID,
  network_name VARCHAR(255) NOT NULL,
  FOREIGN KEY (container_id) REFERENCES containers(id) ON DELETE CASCADE,
  FOREIGN KEY (service_id) REFERENCES services(id) ON DELETE CASCADE,
  CHECK (
    (container_id IS NOT NULL AND service_id IS NULL) OR
    (container_id IS NULL AND service_id IS NOT NULL)
  )
);
```

### Volumes
```sql
CREATE TABLE volumes (
  id UUID PRIMARY KEY,
  container_id UUID,
  service_id UUID,
  volume_mapping VARCHAR(255) NOT NULL,
  FOREIGN KEY (container_id) REFERENCES containers(id) ON DELETE CASCADE,
  FOREIGN KEY (service_id) REFERENCES services(id) ON DELETE CASCADE,
  CHECK (
    (container_id IS NOT NULL AND service_id IS NULL) OR
    (container_id IS NULL AND service_id IS NOT NULL)
  )
);
```

## API Endpoints

### Authentication
```
POST /api/auth/login
POST /api/auth/logout
POST /api/auth/register
GET  /api/auth/me
```

### Hosts
```
GET    /api/hosts
GET    /api/hosts/:id
POST   /api/hosts
PUT    /api/hosts/:id
DELETE /api/hosts/:id
GET    /api/hosts/:id/containers
GET    /api/hosts/:id/services
```

### Clusters
```
GET    /api/clusters
GET    /api/clusters/:id
POST   /api/clusters
PUT    /api/clusters/:id
DELETE /api/clusters/:id
GET    /api/clusters/:id/nodes
GET    /api/clusters/:id/stacks
GET    /api/clusters/:id/services
```

### Stacks
```
GET    /api/stacks
GET    /api/stacks/:id
POST   /api/stacks
PUT    /api/stacks/:id
DELETE /api/stacks/:id
GET    /api/stacks/:id/services
POST   /api/stacks/:id/services
```

### Services
```
GET    /api/services
GET    /api/services/:id
POST   /api/services
PUT    /api/services/:id
DELETE /api/services/:id
GET    /api/services/:id/environment
POST   /api/services/:id/environment
GET    /api/services/:id/networks
POST   /api/services/:id/networks
GET    /api/services/:id/volumes
POST   /api/services/:id/volumes
```

### Containers
```
GET    /api/containers
GET    /api/containers/:id
POST   /api/containers
PUT    /api/containers/:id
DELETE /api/containers/:id
GET    /api/containers/:id/environment
POST   /api/containers/:id/environment
GET    /api/containers/:id/networks
POST   /api/containers/:id/networks
GET    /api/containers/:id/volumes
POST   /api/containers/:id/volumes
```

### Port Management
```
POST   /api/ports/generate
GET    /api/ports/check/:port
GET    /api/ports/available
GET    /api/ports/used
```

## Data Transfer Objects (DTOs)

### Host DTO
```typescript
interface HostDTO {
  id: string;
  name: string;
  location?: string;
  fqdn?: string;
  ip?: string;
  owner?: string;
  isClusterNode: boolean;
  clusterName?: string;
  clusterRole?: 'manager' | 'worker';
  description?: string;
  createdAt: string;
  updatedAt: string;
}
```

### Cluster DTO
```typescript
interface ClusterDTO {
  id: string;
  name: string;
  location?: string;
  owner?: string;
  hasVIP: boolean;
  vipAddress?: string;
  vipFQDN?: string;
  description?: string;
  nodeCount: number;
  createdAt: string;
  updatedAt: string;
}
```

### Stack DTO
```typescript
interface StackDTO {
  id: string;
  name: string;
  host?: string;
  cluster?: string;
  description?: string;
  link?: string;
  services: ServiceDTO[];
  createdAt: string;
  updatedAt: string;
}
```

### Service DTO
```typescript
interface ServiceDTO {
  id: string;
  name: string;
  stackName?: string;
  host?: string;
  image?: string;
  port?: number;
  containerPort?: number;
  link?: string;
  description?: string;
  envVars: EnvironmentVariableDTO[];
  networks: string[];
  volumes: string[];
  createdAt: string;
  updatedAt: string;
}
```

### Container DTO
```typescript
interface ContainerDTO {
  id: string;
  name: string;
  host: string;
  image?: string;
  port?: number;
  containerPort?: number;
  exposure: 'internal' | 'host' | 'lan' | 'wan';
  lanDNS?: string;
  wanDNS?: string;
  proxyURL?: string;
  link?: string;
  owner?: string;
  dockerOptions?: string;
  description?: string;
  envVars: EnvironmentVariableDTO[];
  networks: string[];
  volumes: string[];
  createdAt: string;
  updatedAt: string;
}
```

### Environment Variable DTO
```typescript
interface EnvironmentVariableDTO {
  key: string;
  value: string;
}
```

## Implementation Considerations

### Backend Technology Stack Options
1. **Node.js with Express**
   - TypeORM or Prisma for database access
   - JWT for authentication
   - PostgreSQL or MySQL for the database

2. **Spring Boot (Java)**
   - Spring Data JPA for database access
   - Spring Security for authentication
   - PostgreSQL or MySQL for the database

3. **Django (Python)**
   - Django REST Framework for API
   - Django ORM for database access
   - PostgreSQL for the database

### Frontend Integration
1. Update the Vue app to use Axios or Fetch API for communication with backend
2. Implement authentication with JWT
3. Replace localStorage persistence with API calls
4. Add loading indicators for API operations
5. Implement error handling for API failures

### Security Considerations
1. Implement proper authentication and authorization
2. Use HTTPS in production
3. Input validation on both client and server
4. Rate limiting on API endpoints
5. CORS configuration
6. SQL injection protection
7. Audit logging for critical operations

### Additional Features to Consider
1. User management and role-based access control
2. Port reservation and conflict resolution
3. Audit history for changes to entities
4. Integration with Docker API for container status
5. Automated port scanning to verify usage
6. Search and filtering capabilities
7. Export/import functionality for backup
8. Notifications for important events

## Example Integration Code

### API Service (Vue.js)
```typescript
// src/services/api.service.ts
import axios from 'axios';

const API_URL = process.env.VUE_APP_API_URL || 'http://localhost:3000/api';

const apiClient = axios.create({
  baseURL: API_URL,
  headers: {
    'Content-Type': 'application/json',
  },
});

// Request interceptor for adding auth token
apiClient.interceptors.request.use(
  (config) => {
    const token = localStorage.getItem('token');
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
  },
  (error) => Promise.reject(error)
);

export default {
  // Hosts
  getHosts() {
    return apiClient.get('/hosts');
  },
  getHost(id) {
    return apiClient.get(`/hosts/${id}`);
  },
  createHost(host) {
    return apiClient.post('/hosts', host);
  },
  updateHost(id, host) {
    return apiClient.put(`/hosts/${id}`, host);
  },
  deleteHost(id) {
    return apiClient.delete(`/hosts/${id}`);
  },
  
  // Similar methods for other entities...
  
  // Port generation
  generatePort() {
    return apiClient.post('/ports/generate');
  },
  checkPortAvailability(port) {
    return apiClient.get(`/ports/check/${port}`);
  }
};
```

### Store Integration (Vuex)
```typescript
// src/store/modules/hosts.js
import api from '@/services/api.service';

export default {
  namespaced: true,
  
  state: {
    hosts: [],
    loading: false,
    error: null
  },
  
  getters: {
    getHostById: (state) => (id) => {
      return state.hosts.find(host => host.id === id);
    }
  },
  
  mutations: {
    SET_HOSTS(state, hosts) {
      state.hosts = hosts;
    },
    ADD_HOST(state, host) {
      state.hosts.push(host);
    },
    UPDATE_HOST(state, updatedHost) {
      const index = state.hosts.findIndex(h => h.id === updatedHost.id);
      if (index !== -1) {
        state.hosts.splice(index, 1, updatedHost);
      }
    },
    REMOVE_HOST(state, id) {
      state.hosts = state.hosts.filter(h => h.id !== id);
    },
    SET_LOADING(state, status) {
      state.loading = status;
    },
    SET_ERROR(state, error) {
      state.error = error;
    }
  },
  
  actions: {
    async fetchHosts({ commit }) {
      commit('SET_LOADING', true);
      try {
        const response = await api.getHosts();
        commit('SET_HOSTS', response.data);
        commit('SET_ERROR', null);
      } catch (error) {
        commit('SET_ERROR', error.message);
      } finally {
        commit('SET_LOADING', false);
      }
    },
    
    async createHost({ commit }, host) {
      commit('SET_LOADING', true);
      try {
        const response = await api.createHost(host);
        commit('ADD_HOST', response.data);
        commit('SET_ERROR', null);
        return response.data;
      } catch (error) {
        commit('SET_ERROR', error.message);
        throw error;
      } finally {
        commit('SET_LOADING', false);
      }
    },
    
    // Similar actions for update and delete...
  }
};
```