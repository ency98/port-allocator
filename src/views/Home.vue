<!-- src/views/Home.vue -->
<template>
  <div class="home">
    <h2>Port Allocator</h2>
    <div v-if="showDebug" class="debug-info">
      <h3>Debug Info:</h3>
      <div>{{ entries.length }} entries found | Application is running in Dev Mode</div>
      <div><strong>newEntry Data:</strong> {{ JSON.stringify(newEntry) }}</div>
      <div><strong>Reactive State:</strong> selectedEntry={{ selectedEntry }}, editMode={{ editMode }}</div>
      <div><strong>Note:</strong> editMode=false just means you're creating a new entry (not editing an existing one)</div>
      <button @click="debugForm" class="debug-button">Debug Form State</button>
    </div>
    
    <!-- Debug button will be placed in App.vue footer -->

    <!-- Port Generation Section with Tabs for New/Edit -->
    <div class="port-generator">
      <div class="action-tabs">
        <div class="tab-section">
          <h3>Create New Entry</h3>
          <div class="form-row">
            <div class="form-group">
              <select id="new-type" v-model="newEntry.type" required @change="handleTypeChange">
                <option value="" disabled selected>Select an item to create</option>
                <optgroup label="Infrastructure">
                  <option value="host">Host</option>
                  <option value="cluster">Cluster</option>
                </optgroup>
                <optgroup label="Docker">
                  <option value="container">Container</option>
                  <option value="service">Service</option>
                  <option value="stack">Stack</option>
                </optgroup>
              </select>
            </div>
            <button @click="showCreateForm" class="action-button create-btn" :disabled="!newEntry.type">Create</button>
          </div>
        </div>
        
        <div class="tab-section">
          <h3>Edit Existing Entry</h3>
          <div class="form-row">
            <div class="form-group">
              <select id="edit-entry-selector" v-model="selectedEntry">
                <option value="" disabled selected>Select an entry to edit</option>
                <option v-for="entry in entries" :key="entry.id" :value="entry.id">
                  {{ entry.name }} ({{ entry.type }})
                </option>
              </select>
            </div>
            <button @click="handleEditSelection" class="action-button edit-action-btn" :disabled="selectedEntry === 'new' || selectedEntry === ''">Edit</button>
          </div>
        </div>
      </div>
      
      <!-- Entry Form Explanation -->
      <div v-if="showEntryForm" style="background-color: #d1ecf1; color: #0c5460; padding: 10px; margin: 10px 0; border-radius: 4px; text-align: left;">
        <div><strong>{{ editMode ? 'Editing Existing Entry' : 'Creating New Entry' }}</strong></div>
        <div>{{ editMode ? 'Modify the fields below to update this entry.' : 'Fill out the form below to add a new ' + newEntry.type + ' to your collection.' }}</div>
      </div>

      <!-- Entry Form Section -->
      <div v-if="showEntryForm" class="entry-form">

        <!-- Context-aware host label based on type -->
        <div class="form-group" v-if="(newEntry.type === 'container') || (newEntry.type === 'service' && newEntry.stackName !== 'new_stack')">
          <label for="host">
            {{ newEntry.type === 'container' ? 'Host Name:' : 'Host/Cluster Name:' }}
          </label>
          
          <!-- Changed to select dropdown for better organization -->
          <select id="host" v-model="newEntry.host" @change="checkDuplicateName" class="host-select">
            <option value="" disabled selected>Select a host/cluster</option>
            
            <!-- Hosts section -->
            <optgroup v-if="availableHosts.length > 0" label="Hosts">
              <option v-for="host in availableHosts" :key="host" :value="host">
                {{ host }}
              </option>
            </optgroup>
            
            <!-- Clusters section -->
            <optgroup v-if="availableClusters.length > 0" label="Clusters">
              <option v-for="cluster in availableClusters" :key="cluster" :value="cluster">
                {{ cluster }}
              </option>
            </optgroup>
            
            <!-- Manual entry option -->
            <optgroup label="Other">
              <option value="manual_entry">+ Enter Custom Host/Cluster</option>
            </optgroup>
          </select>
          
          <!-- Input field appears when "Enter Custom Host/Cluster" is selected -->
          <div v-if="newEntry.host === 'manual_entry'" class="custom-host-input">
            <input
              id="customHost"
              v-model="customHostName"
              placeholder="Enter host/cluster name"
              @input="updateCustomHost"
            />
            <div class="input-hint">
              Consider adding this as a proper host/cluster entry for future use
            </div>
          </div>
        </div>

        <!-- Host-specific fields -->
        <template v-if="newEntry.type === 'host'">
          <div class="form-group">
            <label for="name">Host Name:</label>
            <div class="input-with-dropdown">
              <input
                id="name"
                v-model="newEntry.name"
                required
                list="existing-host-names"
                placeholder="e.g. srv-web-01"
                @input="checkDuplicateName"
                @blur="checkDuplicateName"
              />
              <datalist id="existing-host-names">
                <option v-for="name in existingHostNames" :key="name" :value="name"></option>
              </datalist>
            </div>
            <p v-if="isDuplicateName" class="error-message">
              This host name already exists. Please choose another name.
            </p>
          </div>

          <div class="form-group">
            <label for="location">Location:</label>
            <div class="input-with-dropdown">
              <input 
                id="location" 
                v-model="newEntry.location" 
                list="existing-locations"
                placeholder="e.g. Data Center 1" 
              />
              <datalist id="existing-locations">
                <option v-for="location in availableLocations" :key="location" :value="location"></option>
              </datalist>
            </div>
          </div>

          <div class="form-group">
            <label for="fqdn">FQDN:</label>
            <input id="fqdn" v-model="newEntry.fqdn" placeholder="e.g. srv-web-01.example.com" />
          </div>

          <div class="form-group">
            <label for="ip">IP Address:</label>
            <input id="ip" v-model="newEntry.ip" placeholder="e.g. 192.168.1.10" />
          </div>

          <div class="form-group">
            <label for="owner">Owner:</label>
            <input id="owner" v-model="newEntry.owner" placeholder="e.g. Web Team" />
          </div>

          <div class="form-group">
            <label class="checkbox-label">
              <input type="checkbox" v-model="newEntry.isClusterNode" />
              Cluster Node
            </label>
          </div>

          <div v-if="newEntry.isClusterNode" class="form-group cluster-node-details">
            <div class="form-group">
              <label for="clusterName">Cluster Name:</label>
              <select 
                id="clusterName" 
                v-model="newEntry.clusterName"
                class="host-select"
              >
                <option value="" disabled selected>Select a cluster</option>
                
                <!-- Clusters organized by location if available -->
                <template v-if="groupedClusters.length > 0">
                  <optgroup v-for="group in groupedClusters" :key="group.location" :label="group.location">
                    <option v-for="cluster in group.clusters" :key="cluster.id" :value="cluster.name">
                      {{ cluster.name }}
                    </option>
                  </optgroup>
                </template>
                
                <!-- Show ungrouped clusters if any -->
                <template v-if="ungroupedClusters.length > 0">
                  <optgroup label="Other Clusters">
                    <option v-for="cluster in ungroupedClusters" :key="cluster.id" :value="cluster.name">
                      {{ cluster.name }}
                    </option>
                  </optgroup>
                </template>
                
                <!-- Custom entry option -->
                <optgroup label="Other">
                  <option value="manual_entry">+ Enter Custom Cluster</option>
                </optgroup>
              </select>
              
              <!-- Input field appears when custom entry is selected -->
              <div v-if="newEntry.clusterName === 'manual_entry'" class="custom-host-input">
                <input
                  id="customClusterName"
                  v-model="customClusterName"
                  placeholder="Enter cluster name"
                  @input="updateCustomCluster"
                />
                <div class="input-hint">
                  Consider adding this as a proper cluster entry for future use
                </div>
              </div>
            </div>

            <div class="form-group">
              <label>Cluster Role:</label>
              <div class="radio-group">
                <label class="radio-label">
                  <input type="radio" v-model="newEntry.clusterRole" value="manager" />
                  Manager
                </label>
                <label class="radio-label">
                  <input type="radio" v-model="newEntry.clusterRole" value="worker" />
                  Worker
                </label>
              </div>
            </div>
          </div>
        </template>

        <!-- Cluster-specific fields -->
        <template v-if="newEntry.type === 'cluster'">
          <div class="form-group">
            <label for="name">Cluster Name:</label>
            <div class="input-with-dropdown">
              <input
                id="name"
                v-model="newEntry.name"
                required
                list="existing-cluster-names"
                placeholder="e.g. web-cluster"
                @input="checkDuplicateName"
                @blur="checkDuplicateName"
              />
              <datalist id="existing-cluster-names">
                <option v-for="name in existingClusterNames" :key="name" :value="name"></option>
              </datalist>
            </div>
            <p v-if="isDuplicateName" class="error-message">
              This cluster name already exists. Please choose another name.
            </p>
          </div>

          <div class="form-group">
            <label for="owner">Owner:</label>
            <input id="owner" v-model="newEntry.owner" placeholder="e.g. Web Team" />
          </div>

          <div class="form-group">
            <label for="clusterLocation">Location:</label>
            <div class="input-with-dropdown">
              <input 
                id="clusterLocation" 
                v-model="newEntry.location" 
                list="existing-cluster-locations"
                placeholder="e.g. Data Center 1" 
              />
              <datalist id="existing-cluster-locations">
                <option v-for="location in availableLocations" :key="location" :value="location"></option>
              </datalist>
            </div>
          </div>

          <div class="form-group">
            <label class="checkbox-label">
              <input type="checkbox" v-model="newEntry.hasVIP" />
              Virtual IP
            </label>
          </div>

          <div v-if="newEntry.hasVIP" class="form-group vip-details">
            <div class="form-group">
              <label for="vipAddress">VIP Address:</label>
              <input id="vipAddress" v-model="newEntry.vipAddress" placeholder="e.g. 192.168.1.100" />
            </div>
          </div>

          <div class="form-group">
            <label for="vipFQDN">VIP FQDN:</label>
            <input id="vipFQDN" v-model="newEntry.vipFQDN" placeholder="e.g. web-cluster.example.com" />
          </div>
        </template>

        <!-- Container-specific fields -->
        <template v-if="newEntry.type === 'container'">
          <div class="form-group">
            <label for="name">Container Name:</label>
            <div class="input-with-dropdown">
              <input
                id="name"
                v-model="newEntry.name"
                required
                list="existing-container-names"
                placeholder="Container Name"
                @input="checkDuplicateName"
                @blur="checkDuplicateName"
              />
              <datalist id="existing-container-names">
                <option v-for="name in existingContainerNames" :key="name" :value="name"></option>
              </datalist>
            </div>
            <p v-if="isDuplicateName" class="error-message">
              This container name already exists on host "{{ newEntry.host }}". 
              Please choose another name or host.
            </p>
          </div>

          <div class="form-group">
            <label for="owner">Owner:</label>
            <input id="owner" v-model="newEntry.owner" placeholder="e.g. Web Team" />
          </div>

          <div class="form-group">
            <label for="image">Image Name:</label>
            <input id="image" v-model="newEntry.image" placeholder="e.g. nginx:latest" />
          </div>
          
          <div class="form-group">
            <label for="containerPort">Container Port (Optional):</label>
            <input id="containerPort" type="number" v-model.number="newEntry.containerPort" placeholder="e.g. 80" />
          </div>
          
          <div class="form-group">
            <label>Exposure:</label>
            <div class="radio-group exposure-options">
              <label class="radio-label">
                <input type="radio" v-model="newEntry.exposure" value="internal" />
                Internal
              </label>
              <label class="radio-label">
                <input type="radio" v-model="newEntry.exposure" value="host" />
                Host Only
              </label>
              <label class="radio-label">
                <input type="radio" v-model="newEntry.exposure" value="lan" />
                LAN
              </label>
              <label class="radio-label">
                <input type="radio" v-model="newEntry.exposure" value="wan" />
                WAN
              </label>
            </div>
          </div>
          
          <template v-if="newEntry.exposure === 'lan' || newEntry.exposure === 'wan'">
            <div class="form-group">
              <label for="lanDNS">LAN DNS:</label>
              <input id="lanDNS" v-model="newEntry.lanDNS" placeholder="e.g. app.internal" />
            </div>
          </template>
          
          <template v-if="newEntry.exposure === 'wan'">
            <div class="form-group">
              <label for="wanDNS">WAN DNS:</label>
              <input id="wanDNS" v-model="newEntry.wanDNS" placeholder="e.g. app.example.com" />
            </div>
            
            <div class="form-group">
              <label for="proxyURL">Proxy URL:</label>
              <input id="proxyURL" v-model="newEntry.proxyURL" placeholder="e.g. https://proxy.example.com" />
            </div>
          </template>
          
          <div class="form-group">
            <label for="link">Link URL (Optional):</label>
            <input id="link" v-model="newEntry.link" placeholder="e.g. http://localhost:3000" />
          </div>
          
          <div class="accordion-section">
            <button type="button" class="accordion-toggle" @click="toggleAdvancedOptions">
              Advanced Docker Options
              <span class="toggle-icon">{{ showAdvancedOptions ? '−' : '+' }}</span>
            </button>
            <div v-if="showAdvancedOptions" class="accordion-content">
              <div class="form-group">
                <label>Environment Variables:</label>
                <div v-for="(envVar, index) in newEntry.envVars" :key="index" class="env-var-row">
                  <input 
                    v-model="envVar.key"
                    placeholder="Key"
                    class="env-var-key" 
                  />
                  <input 
                    v-model="envVar.value"
                    placeholder="Value"
                    class="env-var-value" 
                  />
                  <button type="button" @click="removeEnvVar(index)" class="remove-btn">×</button>
                </div>
                <button type="button" @click="addEnvVar" class="add-btn">Add Environment Variable</button>
              </div>
              
              <div class="form-group">
                <label>Networks:</label>
                <div v-for="(network, index) in newEntry.networks" :key="index" class="network-row">
                  <input 
                    v-model="newEntry.networks[index]"
                    placeholder="Network name"
                    class="network-input" 
                  />
                  <button type="button" @click="removeNetwork(index)" class="remove-btn">×</button>
                </div>
                <button type="button" @click="addNetwork" class="add-btn">Add Network</button>
              </div>
              
              <div class="form-group">
                <label>Volumes:</label>
                <div v-for="(volume, index) in newEntry.volumes" :key="index" class="volume-row">
                  <input 
                    v-model="newEntry.volumes[index]"
                    placeholder="host_path:container_path"
                    class="volume-input" 
                  />
                  <button type="button" @click="removeVolume(index)" class="remove-btn">×</button>
                </div>
                <button type="button" @click="addVolume" class="add-btn">Add Volume</button>
              </div>
              
              <div class="form-group">
                <label for="dockerOptions">Additional Docker Options:</label>
                <textarea 
                  id="dockerOptions" 
                  v-model="newEntry.dockerOptions" 
                  rows="3"
                  placeholder="Enter additional docker options"
                ></textarea>
              </div>
            </div>
          </div>
        </template>

        <!-- Service-specific fields -->
        <template v-if="newEntry.type === 'service'">
          <div class="form-group">
            <label for="stackName">Stack Name:</label>
            <select id="stackName" v-model="newEntry.stackName" @change="handleStackChange" class="host-select">
              <option value="">None (Standalone Service)</option>
              <option value="new_stack">+ Create New Stack</option>
              
              <!-- Group stacks by host/cluster for better organization -->
              <template v-if="groupedStacks.length > 0">
                <optgroup v-for="group in groupedStacks" :key="group.host" :label="'On ' + group.host">
                  <option v-for="stack in group.stacks" :key="stack.id" :value="stack.name">
                    {{ stack.name }}
                  </option>
                </optgroup>
              </template>
            </select>
          </div>

          <!-- New Stack Creation (appears when "Create New Stack" is selected) -->
          <div v-if="newEntry.stackName === 'new_stack'" class="new-stack-form">
            <h4>Create New Stack</h4>
            <div class="form-group">
              <label for="newStackName">Stack Name:</label>
              <input
                id="newStackName"
                v-model="newStackData.name"
                required
                placeholder="Stack Name"
                @blur="checkDuplicateStackName"
              />
              <p v-if="isDuplicateStackName" class="error-message">This stack name already exists. Please choose another name.</p>
            </div>

            <div class="form-group">
              <label for="newStackHost">Host/Cluster Name:</label>
              <div class="input-with-dropdown">
                <input 
                  id="newStackHost" 
                  v-model="newStackData.host" 
                  list="existing-new-stack-hosts"
                  @input="checkDuplicateStackName"
                  placeholder="e.g. swarm-cluster-01" 
                />
                <datalist id="existing-new-stack-hosts">
                  <option v-for="host in existingHosts" :key="host" :value="host"></option>
                </datalist>
              </div>
            </div>

            <div class="form-group">
              <label for="newStackDescription">Description (Optional):</label>
              <textarea id="newStackDescription" v-model="newStackData.description" rows="2"></textarea>
            </div>

            <div class="form-actions">
              <button
                @click="debugSaveServiceStack"
                :disabled="!newStackData.name || isDuplicateStackName"
                class="create-stack-btn"
              >
                Create Stack
              </button>
              <button @click="cancelNewStack" class="cancel-btn">Cancel</button>
            </div>
          </div>

          <div class="form-group">
            <label for="name">Service Name:</label>
            <div class="input-with-dropdown">
              <input
                id="name"
                v-model="newEntry.name"
                required
                list="existing-service-names"
                placeholder="Service Name"
                @input="checkDuplicateName"
                @blur="checkDuplicateName"
              />
              <datalist id="existing-service-names">
                <option v-for="name in existingServiceNames" :key="name" :value="name"></option>
              </datalist>
            </div>
            <p v-if="isDuplicateName" class="error-message">
              This service name already exists in the selected stack. 
              Please choose another name or a different stack.
            </p>
          </div>

          <div class="form-group">
            <label for="image">Image Name:</label>
            <input id="image" v-model="newEntry.image" placeholder="e.g. nginx:latest" />
          </div>
          
          <div class="form-group">
            <label for="containerPort">Container Port (Optional):</label>
            <input id="containerPort" type="number" v-model.number="newEntry.containerPort" placeholder="e.g. 80" />
          </div>
          
          <div class="form-group">
            <label for="link">Link URL (Optional):</label>
            <input id="link" v-model="newEntry.link" placeholder="e.g. http://localhost:3000" />
          </div>
        </template>

        <!-- Stack-specific fields -->
        <template v-if="newEntry.type === 'stack'">
          <div class="form-group">
            <label for="name">Stack Name:</label>
            <div class="input-with-dropdown">
              <input
                id="name"
                v-model="newEntry.name"
                required
                list="existing-stack-names"
                placeholder="Stack Name"
                @input="checkDuplicateName"
                @blur="checkDuplicateName"
              />
              <datalist id="existing-stack-names">
                <option v-for="name in existingStackNames" :key="name" :value="name"></option>
              </datalist>
            </div>
            <p v-if="isDuplicateName" class="error-message">
              This stack name already exists on host "{{ newEntry.host }}". 
              Please choose another name or host.
            </p>
          </div>
          
          <div class="form-group">
            <label for="stackHost">Host/Cluster Name:</label>
            
            <!-- Changed to select dropdown for better organization -->
            <select id="stackHost" v-model="newEntry.host" @change="checkDuplicateName" class="host-select">
              <option value="" disabled selected>Select a host/cluster</option>
              
              <!-- Hosts section -->
              <optgroup v-if="availableHosts.length > 0" label="Hosts">
                <option v-for="host in availableHosts" :key="host" :value="host">
                  {{ host }}
                </option>
              </optgroup>
              
              <!-- Clusters section -->
              <optgroup v-if="availableClusters.length > 0" label="Clusters">
                <option v-for="cluster in availableClusters" :key="cluster" :value="cluster">
                  {{ cluster }}
                </option>
              </optgroup>
              
              <!-- Manual entry option -->
              <optgroup label="Other">
                <option value="manual_entry">+ Enter Custom Host/Cluster</option>
              </optgroup>
            </select>
            
            <!-- Input field appears when "Enter Custom Host/Cluster" is selected -->
            <div v-if="newEntry.host === 'manual_entry'" class="custom-host-input">
              <input
                id="customStackHost"
                v-model="customHostName"
                placeholder="Enter host/cluster name"
                @input="updateCustomHost"
              />
              <div class="input-hint">
                Consider adding this as a proper host/cluster entry for future use
              </div>
            </div>
          </div>

          <div v-if="!editMode && !selectedStackFromDropdown" class="stack-services">
            <h4>Services in this Stack:</h4>
            <p class="help-text">You can add services to this stack after creating it</p>
          </div>

          <div v-if="editMode && stackServices.length > 0" class="stack-services">
            <h4>Services in this Stack:</h4>
            <ul class="service-list">
              <li v-for="service in stackServices" :key="service.id">
                {{ service.name }} ({{ service.image || 'No image specified' }})
              </li>
            </ul>
          </div>
          
          <div class="form-group">
            <label for="link">Link URL (Optional):</label>
            <input id="link" v-model="newEntry.link" placeholder="e.g. http://my-stack-dashboard.com" />
          </div>
        </template>

        <!-- Description field - common for all types -->
        <div class="form-group">
          <label for="description">Description (Optional):</label>
          <textarea id="description" v-model="newEntry.description" rows="3"></textarea>
        </div>

        <!-- Modified actions div for the entry form -->
        <div class="actions">
          <!-- Host or Cluster type: just Save and Cancel -->
          <template v-if="newEntry.type === 'host' || newEntry.type === 'cluster'">
            <button
              @click="saveEntry"
              :disabled="!newEntry.name || isDuplicateName"
              class="save-btn" style="font-size: 16px; padding: 10px 15px;"
            >
              {{ editMode ? 'Update Entry' : 'Save New Entry' }}
            </button>
            <button @click="cancelEdit" class="cancel-btn">Cancel</button>
          </template>
          
          <!-- Container/Service type: Port generation workflow -->
          <template v-else-if="newEntry.type !== 'stack'">
            <button @click="generatePort" :disabled="!newEntry.name || !newEntry.type || isDuplicateName" 
              class="generate-btn" style="font-size: 16px; padding: 10px 15px; background-color: #28a745;">
              Step 1: Generate Port
            </button>
            <button
              @click="saveEntry"
              :disabled="!newEntry.name || !newEntry.type || !newEntry.port || isDuplicateName"
              class="save-btn" style="font-size: 16px; padding: 10px 15px;"
            >
              Step 2: {{ editMode ? 'Update Entry' : 'Save New Entry' }}
            </button>
            <button @click="cancelEdit" class="cancel-btn">Cancel</button>
          </template>

          <!-- Stack type: Special stack workflow -->
          <template v-else>
            <!-- Different buttons based on whether we're creating a new stack or editing an existing one -->
            <template v-if="editMode">
              <button
                type="button"
                @click="debugSaveStack"
                :disabled="!newEntry.name || isDuplicateName"
                class="save-btn"
              >
                Update Stack
              </button>
              <button
                type="button"
                @click="addServicesMode"
                class="add-service-btn"
              >
                Add Services
              </button>
              <button @click="cancelEdit" class="cancel-btn">Cancel</button>
            </template>
            <template v-else>
              <button
                type="button"
                @click="debugSaveStack"
                :disabled="!newEntry.name || isDuplicateName"
                class="save-btn"
              >
                Save Stack
              </button>
              <button @click="cancelEdit" class="cancel-btn">Cancel</button>
            </template>
          </template>
        </div>
      </div>

      <div v-if="generatedPort" class="port-result">
        <h3>Generated Port: <span class="port-number">{{ generatedPort }}</span></h3>
        <p v-if="isPortConflict" class="conflict-warning">
          Warning: This port conflicts with existing entries. Another port will be generated.
        </p>
      </div>
    </div>

    <!-- Stack Services Management View (For when a stack is created and we switch to service entry mode) -->
    <div v-if="showStackServicesManager" class="stack-services-manager">
      <h3>Add Services to Stack: <span class="stack-name">{{ currentStack.name }}</span></h3>
      <p v-if="currentStack.description" class="stack-description">{{ currentStack.description }}</p>

      <div class="service-entries">
        <div v-for="(service, index) in stackServiceEntries" :key="index" class="service-entry">
          <div class="service-header">
            <h4>Service #{{ index + 1 }}</h4>
            <button v-if="index > 0" @click="removeServiceEntry(index)" class="remove-service-btn">Remove</button>
          </div>

          <div class="form-group">
            <label :for="'serviceName'+index">Service Name:</label>
            <input
              :id="'serviceName'+index"
              v-model="service.name"
              required
              placeholder="Service Name"
              @blur="checkDuplicateServiceName(service)"
            />
            <p v-if="service.isDuplicate" class="error-message">This service name already exists. Please choose another name.</p>
          </div>

          <div class="form-group">
            <label :for="'serviceImage'+index">Image Name:</label>
            <input :id="'serviceImage'+index" v-model="service.image" placeholder="e.g. nginx:latest" />
          </div>

          <div class="form-group">
            <label :for="'serviceDescription'+index">Description (Optional):</label>
            <textarea :id="'serviceDescription'+index" v-model="service.description" rows="2"></textarea>
          </div>

          <div class="service-port-section">
            <button @click="generateServicePort(service)" :disabled="!service.name || service.isDuplicate" class="generate-btn">
              Generate Port
            </button>
            <div v-if="service.port" class="service-port">
              <span>Port: </span>
              <strong>{{ service.port }}</strong>
            </div>
          </div>
        </div>
      </div>

      <div class="stack-actions">
        <button @click="addServiceEntry" class="add-service-btn">Add Another Service</button>
        <button
          @click="saveStackServices"
          :disabled="!canSaveStackServices"
          class="save-btn"
        >
          Save Services
        </button>
        <button @click="cancelStackServicesManager" class="cancel-btn">Cancel</button>
      </div>
    </div>

    <!-- Tabbed Entries Section -->
    <div class="entries-section">
      <div class="entries-tabs">
        <button 
          @click="activeTab = 'entries'" 
          class="tab-button" 
          :class="{ 'active': activeTab === 'entries' }"
        >
          Saved Entries
        </button>
        <button 
          @click="activeTab = 'hosts'" 
          class="tab-button" 
          :class="{ 'active': activeTab === 'hosts' }"
        >
          Hosts
        </button>
        <button 
          @click="activeTab = 'clusters'" 
          class="tab-button" 
          :class="{ 'active': activeTab === 'clusters' }"
        >
          Clusters
        </button>
      </div>
      
      <!-- Saved Entries Tab -->
      <div v-if="activeTab === 'entries'" class="tab-content">
        <div class="section-header">
          <h3>Saved Entries {{ selectedFilterHost ? '- ' + selectedFilterHost : '' }}</h3>
          <button 
            @click="toggleEntriesSection" 
            class="toggle-btn"
            :class="{ 'expanded': entriesSectionExpanded }"
            :title="entriesSectionExpanded ? 'Collapse All' : 'Expand All'"
          >
            <span class="icon">{{ entriesSectionExpanded ? '−' : '+' }}</span>
            <span class="text">{{ entriesSectionExpanded ? 'Collapse' : 'Expand' }}</span>
          </button>
        </div>
      
      <div v-if="entries.length > 0" class="entries-content" :class="{ 'expanded': entriesSectionExpanded }">
        <div v-if="!entriesSectionExpanded" class="summary-view">
          <p class="summary-text">{{ entries.length }} entries across {{ entriesByHost.length }} hosts</p>
        </div>
        
        <!-- Host/Cluster-specific section -->
        <div v-if="entriesSectionExpanded" v-for="(host, hostIndex) in entriesByHost" :key="hostIndex" class="host-section">
          <div class="host-header" @click="toggleHostExpansion(host.hostName)">
            <div class="host-controls">
              <div class="host-counts">
                <span v-if="host.containers.length > 0" class="count-badge container-badge">{{ host.containers.length }} containers</span>
                <span v-if="host.services.length > 0" class="count-badge service-badge">{{ host.services.length }} services</span>
                <span v-if="Object.keys(host.stacksMap).length > 0" class="count-badge stack-badge">{{ Object.keys(host.stacksMap).length }} stacks</span>
              </div>
              <h4 class="host-title">{{ host.hostName }} <span v-if="host.hostIp">: {{ host.hostIp }}</span></h4>
              <button 
                class="toggle-btn small"
                :class="{ 'expanded': expandedHosts[host.hostName] }"
              >
                {{ expandedHosts[host.hostName] ? '−' : '+' }}
              </button>
            </div>
          </div>
          
          <div v-if="expandedHosts[host.hostName]" class="host-content">
            <!-- Containers Table -->
            <table v-if="host.containers.length > 0" class="entries-table">
              <thead>
                <tr>
                  <th>Container Name</th>
                  <th>Port</th>
                  <th>Container Port</th>
                  <th>Image</th>
                  <th>Actions</th>
                </tr>
              </thead>
              <tbody>
                <tr v-for="entry in host.containers" :key="entry.id">
                  <td>{{ entry.name }}</td>
                  <td><strong>{{ entry.port || '-' }}</strong></td>
                  <td>{{ entry.containerPort || '-' }}</td>
                  <td :title="entry.image">{{ getDisplayImage(entry.image) || '-' }}</td>
                  <td class="actions-cell">
                    <button @click="editEntry(entry)" class="edit-btn">Edit</button>
                    <button @click="deleteEntry(entry.id)" class="delete-btn">Delete</button>
                    <a v-if="entry.link" :href="entry.link" target="_blank" class="link-btn">Link</a>
                  </td>
                </tr>
              </tbody>
            </table>
            
            <!-- Services Table (Standalone services not in a stack) -->
            <table v-if="host.services.length > 0" class="entries-table">
              <thead>
                <tr>
                  <th>Service Name</th>
                  <th>Port</th>
                  <th>Container Port</th>
                  <th>Image</th>
                  <th>Actions</th>
                </tr>
              </thead>
              <tbody>
                <tr v-for="entry in host.services" :key="entry.id">
                  <td>{{ entry.name }}</td>
                  <td><strong>{{ entry.port || '-' }}</strong></td>
                  <td>{{ entry.containerPort || '-' }}</td>
                  <td :title="entry.image">{{ getDisplayImage(entry.image) || '-' }}</td>
                  <td class="actions-cell">
                    <button @click="editEntry(entry)" class="edit-btn">Edit</button>
                    <button @click="deleteEntry(entry.id)" class="delete-btn">Delete</button>
                    <a v-if="entry.link" :href="entry.link" target="_blank" class="link-btn">Link</a>
                  </td>
                </tr>
              </tbody>
            </table>
            
            <!-- Stacks with Nested Services -->
            <div v-for="stackName in Object.keys(host.stacksMap).sort()" :key="stackName" class="stack-container">
              <div 
                v-if="host.stacksMap[stackName].stack" 
                class="stack-header"
                @click="host.stacksMap[stackName].services.length > 1 ? toggleStackExpansion(host.hostName, stackName) : null"
                :class="{ 'expandable': host.stacksMap[stackName].services.length > 1 }"
              >
                <div class="stack-info">
                  <div v-if="host.stacksMap[stackName].services.length > 1" class="service-count-container">
                    <span class="service-count">({{ host.stacksMap[stackName].services.length }} services)</span>
                  </div>
                  <span class="stack-name">Stack: {{ stackName }}</span>
                  <div v-if="host.stacksMap[stackName].services.length > 1" class="toggle-container">
                    <button 
                      class="toggle-btn small"
                      :class="{ 'expanded': isStackExpanded(host.hostName, stackName) }"
                    >
                      {{ isStackExpanded(host.hostName, stackName) ? '−' : '+' }}
                    </button>
                  </div>
                </div>
                <span class="stack-actions">
                  <button @click.stop="editEntry(host.stacksMap[stackName].stack)" class="edit-btn">Edit</button>
                  <button @click.stop="deleteEntry(host.stacksMap[stackName].stack.id)" class="delete-btn">Delete</button>
                  <a v-if="host.stacksMap[stackName].stack.link" @click.stop :href="host.stacksMap[stackName].stack.link" target="_blank" class="link-btn">Link</a>
                </span>
              </div>
              
              <!-- Single Service Stack (Always Shown) -->
              <table v-if="host.stacksMap[stackName].services.length === 1" class="entries-table stack-services-table">
                <thead>
                  <tr>
                    <th>Service Name</th>
                    <th>Port</th>
                    <th>Container Port</th>
                    <th>Image</th>
                    <th>Actions</th>
                  </tr>
                </thead>
                <tbody>
                  <tr v-for="service in host.stacksMap[stackName].services" :key="service.id">
                    <td>{{ service.name }}</td>
                    <td><strong>{{ service.port || '-' }}</strong></td>
                    <td>{{ service.containerPort || '-' }}</td>
                    <td :title="service.image">{{ getDisplayImage(service.image) || '-' }}</td>
                    <td class="actions-cell">
                      <button @click="editEntry(service)" class="edit-btn">Edit</button>
                      <button @click="deleteEntry(service.id)" class="delete-btn">Delete</button>
                      <a v-if="service.link" :href="service.link" target="_blank" class="link-btn">Link</a>
                    </td>
                  </tr>
                </tbody>
              </table>
              
              <!-- Multi-Service Stack (Expandable) -->
              <table 
                v-if="host.stacksMap[stackName].services.length > 1 && isStackExpanded(host.hostName, stackName)" 
                class="entries-table stack-services-table"
              >
                <thead>
                  <tr>
                    <th>Service Name</th>
                    <th>Port</th>
                    <th>Container Port</th>
                    <th>Image</th>
                    <th>Actions</th>
                  </tr>
                </thead>
                <tbody>
                  <tr v-for="service in host.stacksMap[stackName].services" :key="service.id">
                    <td>{{ service.name }}</td>
                    <td><strong>{{ service.port || '-' }}</strong></td>
                    <td>{{ service.containerPort || '-' }}</td>
                    <td :title="service.image">{{ getDisplayImage(service.image) || '-' }}</td>
                    <td class="actions-cell">
                      <button @click="editEntry(service)" class="edit-btn">Edit</button>
                      <button @click="deleteEntry(service.id)" class="delete-btn">Delete</button>
                      <a v-if="service.link" :href="service.link" target="_blank" class="link-btn">Link</a>
                    </td>
                  </tr>
                </tbody>
              </table>
            </div>
          </div>
        </div>
      </div>
      
      <p v-else>No entries saved yet.</p>
      </div>
      
      <!-- Hosts Tab -->
      <div v-if="activeTab === 'hosts'" class="tab-content">
        <div class="section-header">
          <h3>Hosts</h3>
        </div>
        
        <div v-if="hostEntries.length > 0" class="hosts-list">
          <div v-for="host in hostEntries" :key="host.id" class="host-card" @click="filterByHost(host.name)">
            <div class="host-card-header">
              <h4>{{ host.name }}</h4>
              <span v-if="host.ip" class="host-ip">{{ host.ip }}</span>
            </div>
            <div class="host-card-content">
              <div v-if="host.location" class="host-location">
                <span class="field-label">Location:</span> {{ host.location }}
              </div>
              <div v-if="host.fqdn" class="host-fqdn">
                <span class="field-label">FQDN:</span> {{ host.fqdn }}
              </div>
              <div v-if="host.owner" class="host-owner">
                <span class="field-label">Owner:</span> {{ host.owner }}
              </div>
              <div v-if="host.isClusterNode" class="host-cluster">
                <span class="field-label">Cluster:</span> {{ host.clusterName }} 
                <span class="host-role">({{ host.clusterRole }})</span>
              </div>
              <div class="host-usage">
                <span class="count-badge container-badge" 
                      v-if="getContainerCountForHost(host.name) > 0">
                  {{ getContainerCountForHost(host.name) }} containers
                </span>
                <span class="count-badge service-badge" 
                      v-if="getServiceCountForHost(host.name) > 0">
                  {{ getServiceCountForHost(host.name) }} services
                </span>
                <span class="count-badge stack-badge" 
                      v-if="getStackCountForHost(host.name) > 0">
                  {{ getStackCountForHost(host.name) }} stacks
                </span>
              </div>
            </div>
            <div class="host-card-footer">
              <button @click.stop="editEntry(host)" class="edit-btn">Edit</button>
              <button @click.stop="deleteEntry(host.id)" class="delete-btn">Delete</button>
            </div>
          </div>
        </div>
        <p v-else>No hosts saved yet.</p>
      </div>
      
      <!-- Clusters Tab -->
      <div v-if="activeTab === 'clusters'" class="tab-content">
        <div class="section-header">
          <h3>Clusters</h3>
        </div>
        
        <div v-if="clusterEntries.length > 0" class="clusters-list">
          <div v-for="cluster in clusterEntries" :key="cluster.id" class="cluster-card" @click="filterByHost(cluster.name)">
            <div class="cluster-card-header">
              <h4>{{ cluster.name }}</h4>
              <span v-if="cluster.hasVIP && cluster.vipAddress" class="cluster-vip">VIP: {{ cluster.vipAddress }}</span>
            </div>
            <div class="cluster-card-content">
              <div v-if="cluster.location" class="cluster-location">
                <span class="field-label">Location:</span> {{ cluster.location }}
              </div>
              <div v-if="cluster.vipFQDN" class="cluster-fqdn">
                <span class="field-label">FQDN:</span> {{ cluster.vipFQDN }}
              </div>
              <div v-if="cluster.owner" class="cluster-owner">
                <span class="field-label">Owner:</span> {{ cluster.owner }}
              </div>
              <div class="cluster-nodes">
                <span class="field-label">Nodes:</span> {{ getNodeCountForCluster(cluster.name) }}
              </div>
              <div class="cluster-usage">
                <span class="count-badge service-badge" 
                      v-if="getServiceCountForHost(cluster.name) > 0">
                  {{ getServiceCountForHost(cluster.name) }} services
                </span>
                <span class="count-badge stack-badge" 
                      v-if="getStackCountForHost(cluster.name) > 0">
                  {{ getStackCountForHost(cluster.name) }} stacks
                </span>
              </div>
            </div>
            <div class="cluster-card-footer">
              <button @click.stop="editEntry(cluster)" class="edit-btn">Edit</button>
              <button @click.stop="deleteEntry(cluster.id)" class="delete-btn">Delete</button>
            </div>
          </div>
        </div>
        <p v-else>No clusters saved yet.</p>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, onMounted, reactive, computed } from 'vue';
import { useRouter } from 'vue-router';

export default {
  name: 'Home',
  setup() {
    const router = useRouter();
    
    // Add global error handling
    window.addEventListener('error', (event) => {
      console.error('Global error caught:', event.error);
    });
    
    console.log('Home component initialized');
    
    // Mock user for development
    const mockUser = { email: 'dev@example.com' };
    const entries = ref([]);
    const generatedPort = ref(null);
    const isPortConflict = ref(false);
    const selectedEntry = ref('');
    const editMode = ref(false);
    const showEntryForm = ref(false);
    
    // Debug toggle
    const showDebug = ref(false);
    const toggleDebug = () => {
      showDebug.value = !showDebug.value;
    };
    
    // Listen for debug toggle events from App.vue
    onMounted(() => {
      window.addEventListener('toggle-debug', (event) => {
        showDebug.value = event.detail.showDebug;
      });
    });
    
    // Advanced options toggle
    const showAdvancedOptions = ref(false);
    const toggleAdvancedOptions = () => {
      showAdvancedOptions.value = !showAdvancedOptions.value;
    };
    
    // Methods for environment variables
    const addEnvVar = () => {
      newEntry.envVars.push({ key: '', value: '' });
    };
    
    const removeEnvVar = (index) => {
      newEntry.envVars.splice(index, 1);
    };
    
    // Methods for networks
    const addNetwork = () => {
      newEntry.networks.push('');
    };
    
    const removeNetwork = (index) => {
      newEntry.networks.splice(index, 1);
    };
    
    // Methods for volumes
    const addVolume = () => {
      newEntry.volumes.push('');
    };
    
    const removeVolume = (index) => {
      newEntry.volumes.splice(index, 1);
    };
    
    // Show the create form
    const showCreateForm = () => {
      // Make sure a type is selected
      if (!newEntry.type) {
        alert('Please select an item type to create');
        return;
      }
      
      editMode.value = false;
      selectedEntry.value = 'new';
      showEntryForm.value = true;
    };
    
    // Handle edit button click
    const handleEditSelection = () => {
      if (selectedEntry.value && selectedEntry.value !== 'new') {
        const entry = entries.value.find(e => e.id === selectedEntry.value);
        if (entry) {
          editEntry(entry);
          showEntryForm.value = true;
        }
      }
    };
    const editEntryId = ref(null);

    // Added for dynamic form
    const isDuplicateName = ref(false);
    const isDuplicateStackName = ref(false);
    const stacks = ref([]);
    const stackServices = ref([]);
    const showStackServicesManager = ref(false);
    const currentStack = ref({ name: '', description: '', host: '' });
    const stackServiceEntries = ref([]);
    const selectedStackFromDropdown = ref(false);
    const newStackData = reactive({
      name: '',
      host: '',
      description: ''
    });

    // Computed property for validating stack services
    const canSaveStackServices = computed(() => {
      // Check if we have at least one service
      if (stackServiceEntries.value.length === 0) return false;

      // Check if all services have names and ports
      return stackServiceEntries.value.every(service =>
        service.name &&
        service.port &&
        !service.isDuplicate
      );
    });
    
    // Custom host entry support
    const customHostName = ref('');
    
    // Update the host field when custom host is entered
    const updateCustomHost = () => {
      if (customHostName.value.trim() !== '') {
        newEntry.host = customHostName.value.trim();
      }
    };
    
    // Computed list of available hosts (manually entered "host" type entries)
    const availableHosts = computed(() => {
      const hosts = [];
      entries.value.forEach(entry => {
        if (entry.type === 'host') {
          hosts.push(entry.name);
        }
      });
      return hosts.sort();
    });
    
    // Computed list of available clusters (manually entered "cluster" type entries)
    const availableClusters = computed(() => {
      const clusters = [];
      entries.value.forEach(entry => {
        if (entry.type === 'cluster') {
          clusters.push(entry.name);
        }
      });
      return clusters.sort();
    });
    
    // Computed list of all existing hosts and clusters for backward compatibility
    const existingHosts = computed(() => {
      const hosts = new Set();
      
      // Add hosts from entries that have a host field
      entries.value.forEach(entry => {
        if (entry.host) {
          hosts.add(entry.host);
        }
      });
      
      // Add manually entered hosts
      availableHosts.value.forEach(host => hosts.add(host));
      
      // Add manually entered clusters
      availableClusters.value.forEach(cluster => hosts.add(cluster));
      
      return Array.from(hosts);
    });
    
    // Computed list of stacks grouped by host/cluster
    const groupedStacks = computed(() => {
      const hostGroups = {};
      
      // Group stacks by their host
      entries.value.forEach(entry => {
        if (entry.type === 'stack') {
          const host = entry.host || 'Unknown Host';
          
          if (!hostGroups[host]) {
            hostGroups[host] = {
              host: host,
              stacks: []
            };
          }
          
          hostGroups[host].stacks.push(entry);
        }
      });
      
      // Convert to array and sort by host name
      return Object.values(hostGroups)
        .sort((a, b) => a.host.localeCompare(b.host));
    });
    
    // Computed list of locations for host form
    const availableLocations = computed(() => {
      const locations = new Set();
      
      entries.value.forEach(entry => {
        if (entry.location && entry.location.trim() !== '') {
          locations.add(entry.location);
        }
      });
      
      return Array.from(locations).sort();
    });
    
    // Computed list of clusters grouped by location
    const groupedClusters = computed(() => {
      const locationGroups = {};
      
      // Group clusters by their location
      entries.value.forEach(entry => {
        if (entry.type === 'cluster' && entry.location) {
          const location = entry.location;
          
          if (!locationGroups[location]) {
            locationGroups[location] = {
              location: location,
              clusters: []
            };
          }
          
          locationGroups[location].clusters.push(entry);
        }
      });
      
      // Convert to array and sort by location name
      return Object.values(locationGroups)
        .sort((a, b) => a.location.localeCompare(b.location));
    });
    
    // Computed list of clusters without location
    const ungroupedClusters = computed(() => {
      return entries.value.filter(entry => 
        entry.type === 'cluster' && (!entry.location || entry.location.trim() === '')
      );
    });
    
    // Custom cluster entry support
    const customClusterName = ref('');
    
    // Update the clusterName field when custom cluster is entered
    const updateCustomCluster = () => {
      if (customClusterName.value.trim() !== '') {
        newEntry.clusterName = customClusterName.value.trim();
      }
    };
    
    // Computed list of existing container names
    const existingContainerNames = computed(() => {
      const names = new Set();
      entries.value.forEach(entry => {
        if (entry.type === 'container') {
          names.add(entry.name);
        }
      });
      return Array.from(names);
    });
    
    // Computed list of existing service names
    const existingServiceNames = computed(() => {
      const names = new Set();
      entries.value.forEach(entry => {
        if (entry.type === 'service') {
          names.add(entry.name);
        }
      });
      return Array.from(names);
    });
    
    // Computed list of existing stack names
    const existingStackNames = computed(() => {
      const names = new Set();
      entries.value.forEach(entry => {
        if (entry.type === 'stack') {
          names.add(entry.name);
        }
      });
      return Array.from(names);
    });
    
    // Computed list of existing host names
    const existingHostNames = computed(() => {
      const names = new Set();
      entries.value.forEach(entry => {
        if (entry.type === 'host') {
          names.add(entry.name);
        }
      });
      return Array.from(names);
    });
    
    // Computed list of existing cluster names
    const existingClusterNames = computed(() => {
      const names = new Set();
      entries.value.forEach(entry => {
        if (entry.type === 'cluster') {
          names.add(entry.name);
        }
      });
      return Array.from(names);
    });
    
    // Computed list of existing clusters for host form
    const existingClusters = computed(() => {
      const clusters = new Set();
      entries.value.forEach(entry => {
        if (entry.type === 'cluster') {
          clusters.add(entry.name);
        }
      });
      return Array.from(clusters);
    });
    
    // Group entries by host/cluster for display
    const entriesByHost = computed(() => {
      const hosts = {};
      
      // First, collect all hosts
      entries.value.forEach(entry => {
        if (entry.host && !hosts[entry.host]) {
          hosts[entry.host] = {
            hostName: entry.host,
            hostIp: '', // Placeholder for future feature
            containers: [],
            services: [],
            stacks: [],
            stacksMap: {} // For organizing services within stacks
          };
        }
      });
      
      // Then, organize entries by type
      entries.value.forEach(entry => {
        // Skip entries without a host
        if (!entry.host) return;
        
        // Create host entry if it doesn't exist (should not happen)
        if (!hosts[entry.host]) {
          hosts[entry.host] = {
            hostName: entry.host,
            hostIp: '',
            containers: [],
            services: [],
            stacks: [],
            stacksMap: {}
          };
        }
        
        // Add entry to appropriate list
        if (entry.type === 'container') {
          hosts[entry.host].containers.push(entry);
        } else if (entry.type === 'stack') {
          hosts[entry.host].stacks.push(entry);
          // Initialize stack in stacksMap
          if (!hosts[entry.host].stacksMap[entry.name]) {
            hosts[entry.host].stacksMap[entry.name] = {
              stack: entry,
              services: []
            };
          } else {
            hosts[entry.host].stacksMap[entry.name].stack = entry;
          }
        } else if (entry.type === 'service') {
          if (entry.stackName) {
            // Ensure stack exists in map
            if (!hosts[entry.host].stacksMap[entry.stackName]) {
              hosts[entry.host].stacksMap[entry.stackName] = {
                stack: null,
                services: []
              };
            }
            // Add service to its stack
            hosts[entry.host].stacksMap[entry.stackName].services.push(entry);
          } else {
            // Standalone service
            hosts[entry.host].services.push(entry);
          }
        }
      });
      
      // Convert to array and sort alphabetically by hostName
      return Object.values(hosts).sort((a, b) => a.hostName.localeCompare(b.hostName));
    });
    
    // Function to display truncated image name
    const getDisplayImage = (image) => {
      if (!image) return '-';
      
      const lastSlashIndex = image.lastIndexOf('/');
      if (lastSlashIndex === -1) return image;
      
      return image.substring(lastSlashIndex + 1);
    };
    
    // State for tabbed and collapsible UI
    const activeTab = ref('entries');
    const entriesSectionExpanded = ref(false);
    const expandedHosts = ref({});
    const expandedStacks = ref({});
    const selectedFilterHost = ref('');
    
    // Computed property for host entries
    const hostEntries = computed(() => {
      return entries.value.filter(entry => entry.type === 'host');
    });
    
    // Computed property for cluster entries
    const clusterEntries = computed(() => {
      return entries.value.filter(entry => entry.type === 'cluster');
    });
    
    // Filter entries by host name
    const filterByHost = (hostName) => {
      selectedFilterHost.value = hostName;
      activeTab.value = 'entries';
      entriesSectionExpanded.value = true;
      
      // Expand the selected host
      expandedHosts.value = {
        [hostName]: true
      };
    };
    
    // Get container count for a host
    const getContainerCountForHost = (hostName) => {
      return entries.value.filter(
        entry => entry.type === 'container' && 
                (entry.host === hostName || entry.name === hostName)
      ).length;
    };
    
    // Get service count for a host/cluster
    const getServiceCountForHost = (hostName) => {
      return entries.value.filter(
        entry => entry.type === 'service' && 
                (entry.host === hostName || entry.name === hostName)
      ).length;
    };
    
    // Get stack count for a host/cluster
    const getStackCountForHost = (hostName) => {
      return entries.value.filter(
        entry => entry.type === 'stack' && 
                (entry.host === hostName || entry.name === hostName)
      ).length;
    };
    
    // Get node count for a cluster
    const getNodeCountForCluster = (clusterName) => {
      return entries.value.filter(
        entry => entry.type === 'host' && 
                entry.isClusterNode && 
                entry.clusterName === clusterName
      ).length;
    };
    
    // Toggle entry section expansion
    const toggleEntriesSection = () => {
      entriesSectionExpanded.value = !entriesSectionExpanded.value;
    };
    
    // Toggle host expansion
    const toggleHostExpansion = (hostName) => {
      expandedHosts.value = {
        ...expandedHosts.value,
        [hostName]: !expandedHosts.value[hostName]
      };
    };
    
    // Toggle stack expansion
    const toggleStackExpansion = (hostName, stackName) => {
      const key = `${hostName}:${stackName}`;
      expandedStacks.value = {
        ...expandedStacks.value,
        [key]: !expandedStacks.value[key]
      };
    };
    
    // Check if a stack is expanded
    const isStackExpanded = (hostName, stackName) => {
      const key = `${hostName}:${stackName}`;
      return !!expandedStacks.value[key];
    };

    const newEntry = reactive({
      // Basic fields
      name: '',
      type: '',
      image: '',
      host: '',
      description: '',
      port: null,
      containerPort: null, // For mapping container ports
      link: '',  // For direct linking to the service/container
      stackName: '',  // For services that belong to a stack
      
      // Host/Cluster specific fields
      location: '',
      fqdn: '',
      ip: '',
      owner: '',
      isClusterNode: false,
      clusterName: '',
      clusterRole: 'worker', // 'manager' or 'worker'
      
      // Cluster specific fields
      hasVIP: false,
      vipAddress: '',
      vipFQDN: '',
      
      // Docker common fields
      exposure: 'internal', // 'internal', 'host', 'lan', 'wan'
      lanDNS: '',
      wanDNS: '',
      proxyURL: '',
      
      // Advanced Docker options
      envVars: [],       // Array of {key: '', value: ''} objects
      networks: [],      // Array of network names
      volumes: [],       // Array of volume mappings
      dockerOptions: ''  // Additional Docker options as text
    });

    // Load all saved entries from localStorage
    const loadEntries = () => {
      try {
        const storedEntries = localStorage.getItem('entries');
        if (storedEntries) {
          entries.value = JSON.parse(storedEntries);
        } else {
          entries.value = [];
        }
      } catch (error) {
        console.error('Error loading entries:', error);
        entries.value = [];
      }
    };

    // Handle type changes
    const handleTypeChange = () => {
      // Reset the name check when type changes
      isDuplicateName.value = false;

      // Reset the form fields that are specific to types
      newEntry.name = '';
      newEntry.image = '';
      newEntry.host = '';
      customHostName.value = '';

      if (newEntry.type === 'service') {
        newEntry.stackName = '';
        loadStacks(); // Load available stacks for dropdown
      } else if (newEntry.type === 'stack') {
        stackServices.value = [];
      }
    };

    // Handle stack change in the service form
    const handleStackChange = () => {
      if (newEntry.stackName === 'new_stack') {
        // Reset new stack form data
        newStackData.name = '';
        newStackData.host = newEntry.host || ''; // Default to current host
        newStackData.description = '';
        isDuplicateStackName.value = false;
      }
    };

    // Load all stacks for the service dropdown
    const loadStacks = () => {
      try {
        stacks.value = entries.value.filter(entry => entry.type === 'stack');
      } catch (error) {
        console.error('Error loading stacks:', error);
      }
    };

    // Load services that belong to a stack (for editing a stack)
    const loadStackServices = (stackName) => {
      try {
        stackServices.value = entries.value.filter(entry => 
          entry.type === 'service' && entry.stackName === stackName
        );
      } catch (error) {
        console.error('Error loading stack services:', error);
      }
    };

    // Check for duplicate name based on type, name, and host/stack
    const checkDuplicateName = () => {
      if (!newEntry.name) return;

      try {
        let conflicts = [];
        
        if (newEntry.type === 'container') {
          // For containers, check name and host
          conflicts = entries.value.filter(entry => 
            entry.name === newEntry.name && 
            entry.type === 'container' && 
            entry.host === newEntry.host
          );
        } else if (newEntry.type === 'service') {
          // For services, check name within the same stack
          if (newEntry.stackName) {
            conflicts = entries.value.filter(entry => 
              entry.name === newEntry.name && 
              entry.type === 'service' && 
              entry.stackName === newEntry.stackName
            );
          } else {
            // Standalone service - check name and host
            conflicts = entries.value.filter(entry => 
              entry.name === newEntry.name && 
              entry.type === 'service' && 
              entry.host === newEntry.host && 
              !entry.stackName
            );
          }
        } else if (newEntry.type === 'stack') {
          // For stacks, check name on the same host
          conflicts = entries.value.filter(entry => 
            entry.name === newEntry.name && 
            entry.type === 'stack' && 
            entry.host === newEntry.host
          );
        }

        // If there are any matching entries, we have a duplicate
        if (conflicts.length > 0) {
          // If we're editing, it's only a duplicate if it's not the same entry
          if (editMode.value && conflicts.length === 1 && conflicts[0].id === editEntryId.value) {
            isDuplicateName.value = false;
          } else {
            isDuplicateName.value = true;
          }
        } else {
          isDuplicateName.value = false;
        }
      } catch (error) {
        console.error('Error checking name conflict:', error);
      }
    };

    // Check for duplicate stack name on the same host
    const checkDuplicateStackName = () => {
      if (!newStackData.name) return;

      try {
        const conflicts = entries.value.filter(entry => 
          entry.name === newStackData.name && 
          entry.type === 'stack' &&
          entry.host === newStackData.host
        );
        
        isDuplicateStackName.value = conflicts.length > 0;
      } catch (error) {
        console.error('Error checking stack name conflict:', error);
      }
    };

    // Create a new stack when adding a service
    const saveStackFromService = () => {
      try {
        if (!newStackData.name) {
          console.error('Stack name is required');
          alert('Stack name is required');
          return;
        }
        
        console.log('Creating stack from service view:', newStackData.name);
        
        // Basic data structure for all types
        const entryData = {
          id: Date.now().toString(), // Generate a unique ID
          name: newStackData.name,
          type: 'stack',
          host: newStackData.host || '',
          description: newStackData.description || '',
          createdAt: new Date()
        };

        console.log('Stack data from service:', JSON.stringify(entryData, null, 2));

        // Add to entries and save to localStorage
        entries.value.push(entryData);
        localStorage.setItem('entries', JSON.stringify(entries.value));
        console.log('Stack created from service with ID:', entryData.id);

        // Update the service form
        newEntry.stackName = newStackData.name;

        // Load stacks to update dropdown
        loadStacks();

        // Set up the stack services manager view immediately after saving
        currentStack.value = {
          id: entryData.id,
          name: entryData.name,
          description: entryData.description,
          host: entryData.host
        };

        // Initialize with the current service as the first entry
        stackServiceEntries.value = [{
          name: newEntry.name || '',
          image: newEntry.image || '',
          description: newEntry.description || '',
          port: newEntry.port || null,
          isDuplicate: false
        }];

        // Show the stack services manager
        showStackServicesManager.value = true;

        // Reset the main form
        resetForm();
        
        // Show success message
        alert(`Stack "${entryData.name}" created successfully!`);
      } catch (error) {
        console.error('Error in saveStackFromService:', error);
        alert('An unexpected error occurred: ' + error.message);
      }
    };

    // Cancel creating a new stack
    const cancelNewStack = () => {
      newEntry.stackName = '';
    };

    // Generate a random port between 1025 and 9999
    const generatePort = async () => {
      isPortConflict.value = false;

      // Generate a random port
      let port = Math.floor(Math.random() * (9999 - 1025 + 1)) + 1025;

      // Check for conflicts
      const isConflict = await checkPortConflict(port);
      if (isConflict) {
        isPortConflict.value = true;
        // Try again with a different port
        generatePort();
      } else {
        generatedPort.value = port;
        newEntry.port = port;
      }
    };

    // Generate a port for a service
    const generateServicePort = async (service) => {
      try {
        // Generate a random port
        let port = Math.floor(Math.random() * (9999 - 1025 + 1)) + 1025;

        // Check for conflicts
        const isConflict = await checkPortConflict(port);
        if (isConflict) {
          // Try again recursively
          generateServicePort(service);
        } else {
          service.port = port;
        }
      } catch (error) {
        console.error('Error generating service port:', error);
      }
    };

    // Check for duplicate service name within the same stack
    const checkDuplicateServiceName = (service) => {
      if (!service.name) return;

      try {
        // First check within the current stack entries to prevent duplicate entries
        const duplicateInForm = stackServiceEntries.value.filter(s => 
          s !== service && s.name.toLowerCase() === service.name.toLowerCase()
        ).length > 0;
        
        if (duplicateInForm) {
          service.isDuplicate = true;
          return;
        }
        
        // Then check in the entries list, but only for the same stack
        const duplicateInEntries = entries.value.filter(entry => 
          entry.name === service.name && 
          entry.type === 'service' && 
          entry.stackName === currentStack.value.name
        ).length > 0;
        
        service.isDuplicate = duplicateInEntries;
      } catch (error) {
        console.error('Error checking service name conflict:', error);
      }
    };

    // Check if a port conflicts with existing entries
    const checkPortConflict = (port) => {
      try {
        // Check for port conflicts in the entries
        const conflicts = entries.value.filter(entry => entry.port === port);

        // If there are any matching entries, we have a conflict
        if (conflicts.length > 0) {
          // If we're editing, it's only a conflict if it's not the same entry
          if (editMode.value && conflicts.length === 1 && conflicts[0].id === editEntryId.value) {
            return false;
          }
          return true;
        }

        return false;
      } catch (error) {
        console.error('Error checking port conflict:', error);
        return true; // Assume conflict on error to be safe
      }
    };

    // Save a new entry to localStorage
    const saveEntry = () => {
      try {
        console.log('Saving entry of type:', newEntry.type);
        
        // Basic data structure for all types
        const entryData = {
          name: newEntry.name,
          type: newEntry.type,
          description: newEntry.description || ''
        };
        
        // Add host fields for container, service types
        if (newEntry.type === 'container' || newEntry.type === 'service' || newEntry.type === 'stack') {
          entryData.host = newEntry.host || '';
        }
        
        // Add owner field for all entries
        if (newEntry.owner) {
          entryData.owner = newEntry.owner;
        }
        
        // Host-specific fields
        if (newEntry.type === 'host') {
          if (newEntry.location) entryData.location = newEntry.location;
          if (newEntry.fqdn) entryData.fqdn = newEntry.fqdn;
          if (newEntry.ip) entryData.ip = newEntry.ip;
          
          entryData.isClusterNode = newEntry.isClusterNode;
          if (newEntry.isClusterNode) {
            entryData.clusterName = newEntry.clusterName;
            entryData.clusterRole = newEntry.clusterRole;
          }
        }
        
        // Cluster-specific fields
        if (newEntry.type === 'cluster') {
          if (newEntry.location) entryData.location = newEntry.location;
          if (newEntry.vipFQDN) entryData.vipFQDN = newEntry.vipFQDN;
          
          entryData.hasVIP = newEntry.hasVIP;
          if (newEntry.hasVIP && newEntry.vipAddress) {
            entryData.vipAddress = newEntry.vipAddress;
          }
        }
        
        // Handle port field differently based on type
        if (newEntry.type === 'container' || newEntry.type === 'service') {
          // For containers and services, port is required
          if (!newEntry.port) {
            alert('Port is required for containers and services. Please generate a port first.');
            return;
          }
          entryData.port = newEntry.port;
        }

        // Add image for container and service types
        if (newEntry.type === 'container' || newEntry.type === 'service') {
          entryData.image = newEntry.image || '';
        }

        // Add stack reference for services
        if (newEntry.type === 'service') {
          entryData.stackName = newEntry.stackName || null;
        }
        
        // Add container port if specified
        if ((newEntry.type === 'container' || newEntry.type === 'service') && newEntry.containerPort) {
          entryData.containerPort = newEntry.containerPort;
        }
        
        // Add exposure settings for container/service/stack
        if (newEntry.type === 'container' || newEntry.type === 'service' || newEntry.type === 'stack') {
          entryData.exposure = newEntry.exposure;
          
          if (newEntry.exposure === 'lan' || newEntry.exposure === 'wan') {
            if (newEntry.lanDNS) entryData.lanDNS = newEntry.lanDNS;
          }
          
          if (newEntry.exposure === 'wan') {
            if (newEntry.wanDNS) entryData.wanDNS = newEntry.wanDNS;
            if (newEntry.proxyURL) entryData.proxyURL = newEntry.proxyURL;
          }
        }
        
        // Add advanced Docker options if they exist
        if ((newEntry.type === 'container' || newEntry.type === 'service' || newEntry.type === 'stack')) {
          if (newEntry.envVars && newEntry.envVars.length > 0) {
            entryData.envVars = [...newEntry.envVars];
          }
          
          if (newEntry.networks && newEntry.networks.length > 0) {
            entryData.networks = [...newEntry.networks];
          }
          
          if (newEntry.volumes && newEntry.volumes.length > 0) {
            entryData.volumes = [...newEntry.volumes];
          }
          
          if (newEntry.dockerOptions) {
            entryData.dockerOptions = newEntry.dockerOptions;
          }
        }
        
        let savedEntryId;

        if (editMode.value) {
          // Update existing entry
          console.log('Updating existing entry with ID:', editEntryId.value);
          const entryIndex = entries.value.findIndex(e => e.id === editEntryId.value);
          
          if (entryIndex !== -1) {
            // Update the entry
            entries.value[entryIndex] = {
              ...entries.value[entryIndex],
              ...entryData,
              updatedAt: new Date()
            };
            savedEntryId = editEntryId.value;
            
            // Save to localStorage
            localStorage.setItem('entries', JSON.stringify(entries.value));
            console.log('Entry updated successfully');
          } else {
            alert('Error: Entry not found');
            return;
          }
        } else {
          // Create new entry with unique ID
          const newEntryWithId = {
            ...entryData,
            id: Date.now().toString(),
            createdAt: new Date()
          };
          
          // Add to entries array
          entries.value.push(newEntryWithId);
          savedEntryId = newEntryWithId.id;
          
          // Save to localStorage
          localStorage.setItem('entries', JSON.stringify(entries.value));
          console.log('Entry created with ID:', savedEntryId);
        }

        // For stack type, transition to the add services view
        if (newEntry.type === 'stack' && !editMode.value) {
          // Set up the stack services manager view
          const savedEntry = {
            id: savedEntryId,
            name: entryData.name,
            description: entryData.description,
            host: entryData.host
          };
          
          // Set up the stack services manager
          currentStack.value = savedEntry;
          
          // Initialize with one empty service entry
          stackServiceEntries.value = [{
            name: '',
            image: '',
            description: '',
            port: null,
            isDuplicate: false
          }];
          
          // Show the stack services manager
          showStackServicesManager.value = true;
        }
        
        // Reset the form and hide the entry form
        resetForm();
        showEntryForm.value = false;
        
        // Show success message
        alert(`${newEntry.type.charAt(0).toUpperCase() + newEntry.type.slice(1)} ${editMode.value ? 'updated' : 'created'} successfully!`);
      } catch (error) {
        console.error('Error saving entry:', error);
        alert('An unexpected error occurred: ' + error.message);
      }
    };

    // Direct manual stack creation
    const debugSaveStack = () => {
      try {
        console.log('DEBUG: Direct stack save attempt');
        
        // Check that we have the minimum required data
        if (!newEntry.name) {
          alert('Stack name is required');
          return;
        }

        // Create data structure for the stack
        const stackData = {
          id: Date.now().toString(),  // Generate a unique ID
          name: newEntry.name,
          type: 'stack',
          // Note: explicitly not setting port field for stacks
          host: newEntry.host || '',
          description: newEntry.description || '',
          createdAt: new Date()
        };
        
        console.log('Attempting simplified stack creation with data:', JSON.stringify(stackData, null, 2));
        
        // Add to entries array
        entries.value.push(stackData);
        
        // Save to localStorage
        localStorage.setItem('entries', JSON.stringify(entries.value));
        console.log('Stack saved successfully with ID:', stackData.id);
        alert('Stack saved successfully!');
        
        // Handle post-save actions
        if (!editMode.value) {
          // Set up stack services manager for adding services
          currentStack.value = {
            id: stackData.id,
            name: stackData.name,
            description: stackData.description || '',
            host: stackData.host || ''
          };
          
          // Initialize service entries form
          stackServiceEntries.value = [{
            name: '',
            image: '',
            description: '',
            port: null,
            isDuplicate: false
          }];
          
          // Show the services manager
          showStackServicesManager.value = true;
        }
        
        // Reset form
        resetForm();
      } catch (outerError) {
        console.error('Unexpected error in stack save:', outerError);
        alert('An unexpected error occurred');
      }
    };
    
    // Direct stack creation from service view
    const debugSaveServiceStack = () => {
      try {
        console.log('DEBUG: Direct stack creation from service view');
        
        // Check that we have the minimum required data
        if (!newStackData.name) {
          alert('Stack name is required');
          return;
        }
        
        // Create data structure for the stack
        const stackData = {
          id: Date.now().toString(),  // Generate a unique ID
          name: newStackData.name,
          type: 'stack',
          host: newStackData.host || '',
          description: newStackData.description || '',
          createdAt: new Date()
        };
        
        console.log('Attempting simplified service stack creation with data:', JSON.stringify(stackData, null, 2));
        
        // Add to entries array
        entries.value.push(stackData);
        
        // Save to localStorage
        localStorage.setItem('entries', JSON.stringify(entries.value));
        console.log('Stack saved successfully with ID:', stackData.id);
        alert('Stack saved successfully!');
        
        // Update the service form to reference this stack
        newEntry.stackName = newStackData.name;
        
        // Set up stack services manager for adding services
        currentStack.value = {
          id: stackData.id,
          name: stackData.name,
          description: stackData.description || '',
          host: stackData.host || ''
        };
        
        // Initialize service entries form with current service data
        stackServiceEntries.value = [{
          name: newEntry.name || '',
          image: newEntry.image || '',
          description: newEntry.description || '',
          port: newEntry.port || null,
          isDuplicate: false
        }];
        
        // Show the services manager
        showStackServicesManager.value = true;
        
        // Reload stacks for dropdown
        loadStacks();
        
        // Reset the stack form
        newStackData.name = '';
        newStackData.host = '';
        newStackData.description = '';
      } catch (outerError) {
        console.error('Unexpected error in service stack save:', outerError);
        alert('An unexpected error occurred');
      }
    };

    // Add a new service entry form
    const addServiceEntry = () => {
      stackServiceEntries.value.push({
        name: '',
        image: '',
        description: '',
        port: null,
        isDuplicate: false
      });
    };

    // Remove a service entry form
    const removeServiceEntry = (index) => {
      stackServiceEntries.value.splice(index, 1);
    };

    // Save all services for a stack
    const saveStackServices = () => {
      try {
        // Track which services were successfully saved
        const savedServices = [];

        // Save each service
        for (const service of stackServiceEntries.value) {
          if (service.name && service.port && !service.isDuplicate) {
            const serviceData = {
              id: Date.now().toString() + Math.floor(Math.random() * 1000), // Ensure unique ID
              name: service.name,
              type: 'service',
              image: service.image,
              description: service.description,
              port: service.port,
              host: currentStack.value.host,
              stackName: currentStack.value.name,
              createdAt: new Date()
            };

            // Add to entries array
            entries.value.push(serviceData);
            savedServices.push(service.name);
          }
        }

        // Save to localStorage
        localStorage.setItem('entries', JSON.stringify(entries.value));

        // Show success message
        if (savedServices.length > 0) {
          alert(`Successfully added ${savedServices.length} service(s) to stack "${currentStack.value.name}"`);

          // Close the manager and refresh entries
          showStackServicesManager.value = false;
        } else {
          alert('No valid services to save. Please add at least one service with a name and port.');
        }
      } catch (error) {
        console.error('Error saving stack services:', error);
      }
    };

    // Cancel the stack services manager
    const cancelStackServicesManager = () => {
      if (confirm('Are you sure you want to cancel? Any unsaved services will be lost.')) {
        showStackServicesManager.value = false;
        currentStack.value = { name: '', description: '', host: '' };
        stackServiceEntries.value = [];
      }
    };

    // Delete an entry from localStorage
    const deleteEntry = (id) => {
      if (confirm('Are you sure you want to delete this entry?')) {
        try {
          // Find the entry to check its type
          const entryIndex = entries.value.findIndex(entry => entry.id === id);
          
          if (entryIndex !== -1) {
            const entryData = entries.value[entryIndex];

            // If deleting a stack, ask about related services
            if (entryData.type === 'stack') {
              const relatedServices = entries.value.filter(entry => 
                entry.type === 'service' && entry.stackName === entryData.name
              );

              if (relatedServices.length > 0) {
                const deleteServices = confirm(`This stack has ${relatedServices.length} associated service(s). Delete them as well?`);

                if (deleteServices) {
                  // Delete all associated services
                  entries.value = entries.value.filter(entry => 
                    !(entry.type === 'service' && entry.stackName === entryData.name)
                  );
                } else {
                  // Just remove the stack association
                  entries.value.forEach(entry => {
                    if (entry.type === 'service' && entry.stackName === entryData.name) {
                      entry.stackName = null;
                    }
                  });
                }
              }
            }

            // Delete the entry itself
            entries.value.splice(entryIndex, 1);
            
            // Save to localStorage
            localStorage.setItem('entries', JSON.stringify(entries.value));
          }
        } catch (error) {
          console.error('Error deleting entry:', error);
        }
      }
    };

    // Edit an existing entry - updated for stack services
    const editEntry = (entry) => {
      editMode.value = true;
      editEntryId.value = entry.id;
      selectedEntry.value = entry.id;

      // Fill the form with entry data
      newEntry.name = entry.name;
      newEntry.type = entry.type;
      newEntry.image = entry.image || '';
      newEntry.host = entry.host || '';
      newEntry.description = entry.description || '';
      newEntry.port = entry.port;
      newEntry.containerPort = entry.containerPort || null;
      newEntry.link = entry.link || '';

      // Handle stack-specific data
      if (entry.type === 'service') {
        newEntry.stackName = entry.stackName || '';
        loadStacks();
      } else if (entry.type === 'stack') {
        loadStackServices(entry.name);
      }

      generatedPort.value = entry.port;
      
      // Show the entry form
      showEntryForm.value = true;
    };

    // Handle the dropdown selection (now just used for selected entry change)
    const handleEntrySelection = () => {
      // This function is now primarily handled by the Edit button
      // We keep it for potential future use with event handlers
      console.log('Entry selection changed to:', selectedEntry.value);
    };

    // Add services to an existing stack
    const addServicesMode = () => {
      if (!newEntry.name) return;
      
      // Set up the stack services manager view to add services to this stack
      currentStack.value = {
        id: editEntryId.value,
        name: newEntry.name,
        description: newEntry.description,
        host: newEntry.host
      };

      // Initialize with one empty service entry
      stackServiceEntries.value = [{
        name: '',
        image: '',
        description: '',
        port: null,
        isDuplicate: false
      }];

      // Show the stack services manager
      showStackServicesManager.value = true;

      // Reset the editing mode
      editMode.value = false;
      editEntryId.value = null;
      selectedEntry.value = 'new';
    };

    // Cancel editing mode
    const cancelEdit = () => {
      editMode.value = false;
      editEntryId.value = null;
      selectedEntry.value = '';
      resetForm();
      showEntryForm.value = false;
    };

    // Reset the form to defaults - updated for new fields
    const resetForm = () => {
      newEntry.name = '';
      newEntry.type = '';
      newEntry.image = '';
      newEntry.host = '';
      newEntry.description = '';
      newEntry.port = null;
      newEntry.containerPort = null;
      newEntry.link = '';
      newEntry.stackName = '';
      newEntry.owner = '';
      newEntry.location = '';
      newEntry.fqdn = '';
      newEntry.ip = '';
      newEntry.isClusterNode = false;
      newEntry.clusterName = '';
      newEntry.clusterRole = 'worker';
      newEntry.hasVIP = false;
      newEntry.vipAddress = '';
      newEntry.vipFQDN = '';
      newEntry.exposure = 'internal';
      newEntry.lanDNS = '';
      newEntry.wanDNS = '';
      newEntry.proxyURL = '';
      newEntry.envVars = [];
      newEntry.networks = [];
      newEntry.volumes = [];
      newEntry.dockerOptions = '';
      
      customHostName.value = '';
      customClusterName.value = '';
      generatedPort.value = null;
      editMode.value = false;
      editEntryId.value = null;
      isDuplicateName.value = false;
      stackServices.value = [];
      showAdvancedOptions.value = false;
    };
    
    // Debug function to diagnose form issues
    const debugForm = () => {
      console.log('============ DEBUG FORM STATE ============');
      console.log('Entries:', entries.value);
      console.log('newEntry object:', newEntry);
      console.log('selectedEntry:', selectedEntry.value);
      console.log('editMode:', editMode.value);
      console.log('Generating Sample Entry');
      
      // Generate a test port
      generatePort();
      
      // Force render the entry form
      selectedEntry.value = 'new';
      
      alert('Debug information logged to console. Check browser developer tools.');
    };

    // Load entries when component mounts
    onMounted(() => {
      console.log('Home component mounted, loading entries');
      loadEntries();
      console.log('After loadEntries call, entries count:', entries.value.length);
      
      // Initialize with a sample entry if no entries exist
      if (entries.value.length === 0) {
        console.log('Creating sample entry for demonstration');
        const sampleEntry = {
          id: 'sample-' + Date.now(),
          name: 'Sample Container',
          type: 'container',
          host: 'localhost',
          description: 'Sample entry for demonstration',
          port: 3000,
          createdAt: new Date()
        };
        entries.value.push(sampleEntry);
        localStorage.setItem('entries', JSON.stringify(entries.value));
      }
    });

    return {
      entries,
      newEntry,
      generatedPort,
      isPortConflict,
      selectedEntry,
      editMode,
      isDuplicateName,
      isDuplicateStackName,
      stacks,
      stackServices,
      newStackData,
      showStackServicesManager,
      currentStack,
      stackServiceEntries,
      canSaveStackServices,
      selectedStackFromDropdown,
      handleTypeChange,
      handleStackChange,
      checkDuplicateName,
      checkDuplicateStackName,
      saveStackFromService,
      debugSaveStack,
      debugSaveServiceStack,
      cancelNewStack,
      generatePort,
      generateServicePort,
      checkDuplicateServiceName,
      saveEntry,
      addServicesMode,
      addServiceEntry,
      removeServiceEntry,
      saveStackServices,
      cancelStackServicesManager,
      deleteEntry,
      editEntry,
      handleEntrySelection,
      cancelEdit,
      debugForm,
      showDebug,
      toggleDebug,
      existingHosts,
      existingContainerNames,
      existingServiceNames,
      existingStackNames,
      existingHostNames,
      existingClusterNames,
      existingClusters,
      availableHosts,
      availableClusters,
      availableLocations,
      groupedStacks,
      groupedClusters,
      ungroupedClusters,
      customHostName,
      customClusterName,
      updateCustomHost,
      updateCustomCluster,
      entriesByHost,
      getDisplayImage,
      showEntryForm,
      showCreateForm,
      handleEditSelection,
      activeTab,
      entriesSectionExpanded,
      expandedHosts,
      expandedStacks,
      selectedFilterHost,
      hostEntries,
      clusterEntries,
      toggleEntriesSection,
      toggleHostExpansion,
      toggleStackExpansion,
      isStackExpanded,
      filterByHost,
      getContainerCountForHost,
      getServiceCountForHost,
      getStackCountForHost,
      getNodeCountForCluster,
      // Advanced options
      showAdvancedOptions,
      toggleAdvancedOptions,
      addEnvVar,
      removeEnvVar,
      addNetwork,
      removeNetwork,
      addVolume,
      removeVolume
    };
  }
}
</script>

<style scoped>
.home {
  max-width: 1000px;
  margin: 0 auto;
}

h2, h3 {
  color: var(--text-color, #2c3e50);
}

.port-generator {
  background-color: var(--card-bg, #f9f9f9);
  padding: 20px;
  border-radius: 8px;
  margin-bottom: 30px;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
  transform: translateZ(0);
  position: relative;
  border: 1px solid var(--bg2, rgba(0, 0, 0, 0.05));
  border-top: 4px solid var(--green, #b8bb26);
}

@media (max-width: 768px) {
  .port-generator {
    padding: 15px;
  }
}

@media (max-width: 480px) {
  .port-generator {
    padding: 10px;
  }
}

body.light-theme .port-generator {
  border-top: 4px solid var(--green-dim, #98971a);
}

.entries-section {
  margin-top: 30px;
  background-color: var(--card-bg, #f9f9f9);
  padding: 20px;
  border-radius: 8px;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
  transform: translateZ(0);
  position: relative;
  border: 1px solid var(--bg2, rgba(0, 0, 0, 0.05));
  border-top: 4px solid var(--blue, #83a598);
}

body.light-theme .entries-section {
  border-top: 4px solid var(--blue-dim, #458588);
}

/* For the heading to stand out */
.port-generator h2,
.entries-section h3 {
  margin-top: 0;
  border-bottom: 1px solid var(--bg2, rgba(0, 0, 0, 0.1));
  padding-bottom: 10px;
  margin-bottom: 20px;
}

.entry-form {
  background-color: var(--bg1, #fff);
  padding: 15px;
  border-radius: 6px;
  margin-top: 15px;
  border: 1px solid var(--input-border, #ddd);
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.08);
}

.port-result {
  margin-top: 20px;
  padding: 15px;
  background-color: var(--result-bg, #e8f5e9);
  border-radius: 6px;
  border-left: 8px solid var(--result-border, #4CAF50);
  border-right: 8px solid var(--result-border, #4CAF50);
  box-shadow: 0 3px 8px rgba(0, 0, 0, 0.15);
  font-weight: bold;
}

.port-number {
  font-size: 28px;
  font-weight: bold;
  color: var(--port-number-color, #4CAF50);
  text-shadow: 0 1px 2px rgba(0, 0, 0, 0.1);
  background-color: rgba(0, 0, 0, 0.05);
  padding: 2px 10px;
  border-radius: 4px;
}

.conflict-warning {
  color: var(--conflict-color, #f44336);
  font-weight: bold;
}

.host-section {
  margin-bottom: 30px;
}

.host-header {
  background-color: var(--header-bg, #ebdbb2);
  padding: 10px 15px;
  border-radius: 4px;
  margin-bottom: 15px;
  border-left: 4px solid var(--blue, #076678);
  border-right: 4px solid var(--blue, #076678);
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

/* Host title now moved to .host-title class */

.stack-container {
  margin: 20px 0;
  border-left: 3px solid var(--yellow-dim, #d79921);
  padding-left: 15px;
}

.stack-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  background-color: var(--bg2, #d5c4a1);
  padding: 8px 15px;
  border-radius: 4px;
  margin-bottom: 10px;
}

/* Stack name styles now merged with other .stack-name definition */

.stack-actions {
  display: flex;
  gap: 5px;
}

.stack-services-table {
  margin-left: 20px;
  width: calc(100% - 20px);
}

.link-btn {
  display: inline-block;
  background-color: var(--blue, #076678);
  color: white;
  padding: 5px 10px;
  text-decoration: none;
  border-radius: 4px;
  font-size: 0.9em;
  margin-left: 5px;
}

.link-btn:hover {
  background-color: var(--blue-dim, #458588);
  color: white;
}

/* Tabbed UI Styles */
.entries-tabs {
  display: flex;
  border-bottom: 1px solid var(--bg3, #bdae93);
  margin-bottom: 20px;
  width: 100%;
}

.tab-button {
  flex: 1;
  padding: 10px 20px;
  background-color: transparent;
  border: none;
  border-bottom: 3px solid transparent;
  color: var(--fg3, #bdae93);
  cursor: pointer;
  font-weight: bold;
  transition: all 0.3s ease;
  text-align: center;
}

@media (max-width: 768px) {
  .entries-tabs {
    flex-wrap: nowrap;
    gap: 0;
  }
  
  .tab-button {
    padding: 8px 5px;
    font-size: 14px;
    min-width: 0;
  }
}

@media (max-width: 480px) {
  .entries-tabs {
    flex-wrap: nowrap;
    gap: 0;
    border-bottom: 1px solid var(--bg3, #bdae93);
  }
  
  .tab-button {
    padding: 8px 2px;
    font-size: 12px;
  }
  
  .tab-button.active {
    border-bottom-width: 2px;
  }
}

.tab-button:hover {
  border-bottom-color: var(--bg4, #a89984);
  color: var(--fg2, #d5c4a1);
}

.tab-button.active {
  border-bottom-color: var(--blue, #83a598);
  color: var(--fg, #fbf1c7);
}

body.light-theme .tab-button.active {
  border-bottom-color: var(--blue-dim, #458588);
  color: var(--fg, #282828);
}

.tab-content {
  padding: 5px;
}

/* Host and Cluster Cards */
.hosts-list, .clusters-list {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(300px, 1fr));
  gap: 20px;
  margin-top: 20px;
}

@media (max-width: 480px) {
  .hosts-list, .clusters-list {
    grid-template-columns: 1fr;
  }
}

.host-card, .cluster-card {
  background-color: var(--bg1, #3c3836);
  border-radius: 8px;
  overflow: hidden;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
  transition: transform 0.2s, box-shadow 0.2s;
  cursor: pointer;
}

.host-card:hover, .cluster-card:hover {
  transform: translateY(-5px);
  box-shadow: 0 5px 15px rgba(0, 0, 0, 0.15);
}

.host-card-header, .cluster-card-header {
  background-color: var(--bg2, #504945);
  padding: 15px;
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.host-card-header h4, .cluster-card-header h4 {
  margin: 0;
  font-size: 16px;
  color: var(--fg, #fbf1c7);
}

.host-ip, .cluster-vip {
  font-size: 12px;
  color: var(--fg3, #bdae93);
}

.host-card-content, .cluster-card-content {
  padding: 15px;
}

.field-label {
  font-weight: bold;
  color: var(--fg2, #d5c4a1);
}

.host-location, .host-fqdn, .host-owner, .host-cluster,
.cluster-location, .cluster-fqdn, .cluster-owner, .cluster-nodes {
  margin-bottom: 8px;
  font-size: 14px;
}

.host-role {
  font-style: italic;
  color: var(--fg3, #bdae93);
}

.host-usage, .cluster-usage {
  margin-top: 12px;
  display: flex;
  flex-wrap: wrap;
  gap: 5px;
}

.host-card-footer, .cluster-card-footer {
  background-color: var(--bg2, #504945);
  padding: 10px 15px;
  display: flex;
  justify-content: flex-end;
  gap: 10px;
}

/* Collapsible UI Styles */
.section-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 15px;
}

.section-header h3 {
  text-align: left;
}

.toggle-btn {
  background-color: var(--button-secondary, #9e9e9e);
  color: white;
  border: none;
  border-radius: 4px;
  padding: 5px 10px;
  cursor: pointer;
  display: flex;
  align-items: center;
  gap: 5px;
  transition: background-color 0.2s;
}

.toggle-btn.expanded {
  background-color: var(--button-primary, #2196F3);
  box-shadow: 0 0 4px var(--button-primary, #2196F3);
}

.toggle-btn:hover {
  opacity: 0.9;
}

.toggle-btn.small {
  padding: 2px 6px;
  font-size: 12px;
  min-width: 24px;
  min-height: 24px;
  display: inline-flex;
  justify-content: center;
  align-items: center;
  border-radius: 50%;
  background-color: var(--bg3, #bdae93);
}

.entries-content:not(.expanded) {
  border: 1px solid var(--bg2, rgba(0, 0, 0, 0.1));
  border-radius: 4px;
  padding: 15px;
}

.summary-view {
  text-align: center;
  padding: 10px;
}

.summary-text {
  color: var(--text-color, #333);
  font-style: italic;
}

.host-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  cursor: pointer;
  transition: background-color 0.2s;
}

.host-header:hover {
  background-color: var(--bg1, rgba(255, 255, 255, 0.1));
}

.host-controls {
  display: flex;
  align-items: center;
  width: 100%;
  gap: 20px;
}

.host-counts {
  display: flex;
  gap: 10px;
  align-items: center; /* Ensure badges are vertically centered */
}

.host-title {
  margin: 0;
  color: var(--text-color, #333);
  font-size: 16px;
  flex: 1;
  text-align: center;
}

.count-badge {
  background-color: var(--bg3, #bdae93);
  color: white;
  padding: 4px 10px;
  font-size: 12px;
  border-radius: 12px;
  display: inline-flex;
  align-items: center;
  justify-content: center;
}

.container-badge {
  background-color: var(--blue, #076678);
}

.service-badge {
  background-color: var(--green, #79740e);
}

.stack-badge {
  background-color: var(--yellow, #b57614);
}

.host-content {
  padding-left: 15px;
  margin-top: 10px;
  border-left: 2px solid var(--bg2, rgba(0, 0, 0, 0.1));
}

.stack-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 8px 15px;
  border-radius: 4px;
  margin-bottom: 10px;
}

.stack-header.expandable {
  cursor: pointer;
}

.stack-header.expandable:hover {
  background-color: var(--bg2, rgba(0, 0, 0, 0.05));
}

.stack-info {
  display: flex;
  align-items: center;
  gap: 15px;
}

.stack-name {
  flex: 1;
  text-align: center;
  font-weight: bold;
  color: var(--text-color, #333);
}

.toggle-container {
  display: flex;
  align-items: center;
}

.service-count-container {
  display: flex;
  align-items: center;
  gap: 12px;
}

.service-count {
  font-size: 13px;
  color: var(--fg3, #666);
}

.entries-table {
  width: 100%;
  border-collapse: collapse;
  margin-top: 15px;
}

.entries-table th, .entries-table td {
  border: 1px solid var(--table-border, #ddd);
  padding: 12px;
  text-align: left;
}

@media (max-width: 768px) {
  .entries-table, .entries-table thead, .entries-table tbody, .entries-table th, .entries-table td, .entries-table tr {
    display: block;
  }
  
  .entries-table thead tr {
    position: absolute;
    top: -9999px;
    left: -9999px;
  }
  
  .entries-table tr {
    border: 1px solid var(--table-border, #ddd);
    margin-bottom: 15px;
  }
  
  .entries-table td {
    border: none;
    border-bottom: 1px solid var(--table-border, #ddd);
    position: relative;
    padding-left: 50%;
    min-height: 30px;
  }
  
  .entries-table td:before {
    position: absolute;
    top: 12px;
    left: 12px;
    width: 45%;
    padding-right: 10px;
    white-space: nowrap;
    font-weight: bold;
  }
  
  /* Add data labels for each table cell */
  .entries-table td:nth-of-type(1):before { content: "Name"; }
  .entries-table td:nth-of-type(2):before { content: "Port"; }
  .entries-table td:nth-of-type(3):before { content: "Container Port"; }
  .entries-table td:nth-of-type(4):before { content: "Image"; }
  .entries-table td:nth-of-type(5):before { content: "Actions"; }
  
  .actions-cell {
    display: flex;
    flex-wrap: wrap;
    gap: 5px;
    justify-content: flex-end;
  }
}

.entries-table th {
  background-color: var(--table-header-bg, #f2f2f2);
  font-weight: bold;
}

.entries-table tr:nth-child(even) {
  background-color: var(--table-row-even, #f9f9f9);
}

.entries-table tr:hover {
  background-color: var(--table-row-hover, #f1f8e9);
}

.form-group {
  margin-bottom: 15px;
}

.form-group label {
  display: block;
  margin-bottom: 5px;
  font-weight: bold;
  color: var(--text-color, inherit);
}

input, select, textarea {
  width: 100%;
  padding: 10px;
  border: 1px solid var(--input-border, #ddd);
  border-radius: 4px;
  font-size: 14px;
  background-color: var(--input-bg, white);
  color: var(--text-color, #333);
}

.input-with-dropdown {
  position: relative;
  width: 100%;
}

.input-with-dropdown input {
  padding-right: 25px;
  background-image: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='12' height='12' viewBox='0 0 24 24'%3E%3Cpath fill='%23666' d='M7 10l5 5 5-5z'/%3E%3C/svg%3E");
  background-repeat: no-repeat;
  background-position: right 10px center;
}

.actions {
  display: flex;
  gap: 10px;
  margin-top: 20px;
}

@media (max-width: 768px) {
  .actions {
    flex-direction: column;
  }
  
  .actions button {
    width: 100%;
    margin-bottom: 5px;
  }
}

.action-tabs {
  display: flex;
  flex-direction: column;
  gap: 20px;
  margin-bottom: 20px;
}

.tab-section {
  background-color: var(--bg1, #fff);
  padding: 15px;
  border-radius: 6px;
  border: 1px solid var(--input-border, #ddd);
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.08);
}

.tab-section h3 {
  margin-top: 0;
  margin-bottom: 15px;
  font-size: 18px;
  color: var(--text-color, #333);
  border-bottom: 1px solid var(--bg2, rgba(0, 0, 0, 0.1));
  padding-bottom: 10px;
  text-align: left;
}

.form-row {
  display: flex;
  align-items: flex-end;
  gap: 15px;
}

.form-row .form-group {
  flex: 1;
  margin-bottom: 0;
}

@media (max-width: 768px) {
  .form-row {
    flex-direction: column;
    align-items: stretch;
    gap: 10px;
  }
  
  .form-row .form-group {
    margin-bottom: 10px;
  }
}

.action-button {
  padding: 10px 20px;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-weight: bold;
  transition: background-color 0.3s;
  min-width: 100px;
}

.create-btn {
  background-color: var(--green, #79740e);
  color: white;
}

.create-btn:hover {
  background-color: var(--green-dim, #98971a);
}

.action-button:disabled,
.create-btn:disabled,
.edit-action-btn:disabled {
  background-color: var(--bg3, #bdae93);
  cursor: not-allowed;
  opacity: 0.6;
}

.edit-btn {
  background-color: #fe8019; /* Dark mode orange color */
  color: white;
  padding: 5px 10px;
  margin-right: 5px;
}

.edit-btn:hover {
  background-color: #d65d0e; /* Dark mode orange-dim color */
}

.edit-action-btn {
  background-color: #fe8019; /* Dark mode orange color */
  color: white;
}

.edit-action-btn:hover {
  background-color: #d65d0e; /* Dark mode orange-dim color */
}

.generate-btn {
  background-color: var(--button-primary, #4CAF50);
}

.generate-btn:hover {
  background-color: var(--button-primary-hover, #45a049);
}

.save-btn {
  background-color: var(--button-primary, #2196F3);
}

.save-btn:hover {
  background-color: var(--button-primary-hover, #0b7dda);
}

.cancel-btn {
  background-color: var(--button-secondary, #9e9e9e);
}

.cancel-btn:hover {
  background-color: var(--bg2, #858585);
}

.edit-btn {
  background-color: #fe8019; /* Dark mode orange color */
  color: white;
  padding: 5px 10px;
  margin-right: 5px;
}

.edit-btn:hover {
  background-color: #d65d0e; /* Dark mode orange-dim color */
}

.delete-btn {
  background-color: #fb4934; /* Dark mode red color */
  color: white;
  padding: 5px 10px;
}

.delete-btn:hover {
  background-color: #cc2412; /* Dark mode red-dim color */
}

/* Additional styles for stack and service workflow */
.error-message {
  color: var(--red, #f44336);
  font-size: 0.85em;
  margin-top: 5px;
  margin-bottom: 0;
}

.stack-services {
  margin-top: 15px;
  padding: 15px;
  background-color: var(--bg1, rgba(0,0,0,0.03));
  border-radius: 6px;
}

.stack-services h4 {
  margin-top: 0;
  margin-bottom: 10px;
}

.help-text {
  font-style: italic;
  color: var(--fg3, #666);
  margin: 5px 0;
}

.service-list {
  list-style-type: none;
  padding: 0;
  margin: 0;
}

.service-list li {
  padding: 8px 10px;
  border-bottom: 1px solid var(--bg2, rgba(0,0,0,0.05));
}

.service-list li:last-child {
  border-bottom: none;
}

.new-stack-form {
  background-color: var(--bg1, #f5f5f5);
  padding: 15px;
  margin: 10px 0 20px 0;
  border-radius: 6px;
  border: 1px solid var(--bg2, rgba(0,0,0,0.1));
}

.new-stack-form h4 {
  margin-top: 0;
  margin-bottom: 15px;
  color: var(--text-color, #333);
}

.form-actions {
  display: flex;
  gap: 10px;
  margin-top: 15px;
}

.create-stack-btn {
  background-color: var(--button-primary, #2196F3);
  color: var(--button-text, white);
}

.create-stack-btn:hover {
  background-color: var(--button-primary-hover, #0b7dda);
}

.stack-services-manager {
  background-color: var(--card-bg, #f9f9f9);
  padding: 20px;
  border-radius: 8px;
  margin-top: 20px;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
  border: 1px solid var(--bg2, rgba(0,0,0,0.05));
  border-left: 4px solid var(--yellow, #b57614);
  border-right: 4px solid var(--yellow, #b57614);
}

.stack-services-manager h3 {
  margin-top: 0;
  color: var(--text-color, #333);
}

.stack-name {
  font-weight: bold;
  color: var(--yellow, #d79921);
}

.stack-description {
  font-style: italic;
  color: var(--fg3, #666);
  margin-bottom: 20px;
}

.service-entries {
  margin-bottom: 20px;
}

.service-entry {
  background-color: var(--bg1, #fff);
  padding: 15px;
  border-radius: 6px;
  margin-bottom: 15px;
  border: 1px solid var(--bg2, rgba(0,0,0,0.1));
  border-left: 4px solid var(--blue, #076678);
  border-right: 4px solid var(--blue, #076678);
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.06);
}

.service-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 15px;
}

.service-header h4 {
  margin: 0;
  color: var(--text-color, #333);
}

.remove-service-btn {
  background-color: var(--button-danger, #f44336);
  color: white;
  font-size: 0.85em;
  padding: 5px 10px;
}

.service-port-section {
  display: flex;
  align-items: center;
  margin-top: 10px;
}

.service-port {
  margin-left: 15px;
  font-size: 16px;
}

.service-port strong {
  color: var(--port-number-color, #4CAF50);
  font-size: 18px;
}

.stack-actions {
  display: flex;
  gap: 10px;
}

.add-service-btn {
  background-color: var(--green-dim, #98971a);
  color: white;
}

.add-service-btn:hover {
  background-color: var(--green, #79740e);
}

/* Debug styles moved to App.vue */

.debug-icon.active {
  filter: invert(80%) sepia(30%) saturate(1000%) hue-rotate(340deg) brightness(100%);
}

body.light-theme .debug-icon.active {
  filter: invert(15%) sepia(90%) saturate(2500%) hue-rotate(340deg) brightness(90%);
}

.debug-info {
  background-color: var(--bg1, #3c3836);
  color: var(--fg, #fbf1c7);
  padding: 15px;
  margin-bottom: 20px;
  border-radius: 6px;
  border-left: 4px solid var(--red, #fb4934);
  border-right: 4px solid var(--red, #fb4934);
  text-align: left;
}

.debug-info h3 {
  margin-top: 0;
  border-bottom: 1px solid var(--bg3, #665c54);
  padding-bottom: 5px;
}

.debug-button {
  margin-top: 10px;
  background-color: var(--blue, #83a598);
  color: white;
  border: none;
  padding: 8px 12px;
  border-radius: 4px;
  cursor: pointer;
}

.debug-button:hover {
  background-color: var(--blue-dim, #458588);
}

.auth-status {
  background-color: var(--bg1, #f5f5f5);
  padding: 8px 15px;
  margin-bottom: 15px;
  border-radius: 4px;
  text-align: left;
  border-left: 4px solid var(--button-primary, #2196F3);
}

.auth-status p {
  margin: 0;
  font-size: 14px;
}

/* Styles for new form elements */
.checkbox-label, .radio-label {
  display: inline-flex;
  align-items: center;
  cursor: pointer;
  margin-right: 15px;
}

.checkbox-label input[type="checkbox"], 
.radio-label input[type="radio"] {
  margin-right: 5px;
}

.radio-group {
  display: flex;
  flex-wrap: wrap;
  gap: 10px;
}

.exposure-options {
  margin-top: 5px;
}

@media (max-width: 480px) {
  .radio-group.exposure-options {
    flex-direction: column;
    align-items: flex-start;
  }
  
  .radio-group.exposure-options .radio-label {
    margin-bottom: 8px;
  }
}

.cluster-node-details,
.vip-details {
  margin-left: 15px;
  padding: 10px;
  border-left: 3px solid var(--bg3, #bdae93);
  background-color: var(--bg1, rgba(0,0,0,0.03));
  border-radius: 4px;
}

.accordion-section {
  margin-top: 15px;
  border: 1px solid var(--bg2, rgba(0,0,0,0.1));
  border-radius: 6px;
  overflow: hidden;
}

.accordion-toggle {
  width: 100%;
  text-align: left;
  padding: 10px 15px;
  background-color: var(--bg2, #504945);
  cursor: pointer;
  display: flex;
  justify-content: space-between;
  align-items: center;
  border: none;
  color: var(--fg, #fbf1c7);
  font-weight: bold;
}

.accordion-content {
  padding: 15px;
  background-color: var(--bg1, #3c3836);
}

.toggle-icon {
  font-size: 18px;
  font-weight: bold;
  margin-left: 5px;
}

.env-var-row,
.network-row,
.volume-row {
  display: flex;
  margin-bottom: 5px;
  gap: 5px;
}

.env-var-key {
  width: 30%;
}

.env-var-value {
  width: 60%;
}

@media (max-width: 768px) {
  .env-var-row {
    flex-wrap: wrap;
  }
  
  .env-var-key,
  .env-var-value {
    width: calc(50% - 20px);
  }
}

@media (max-width: 480px) {
  .env-var-row {
    flex-direction: column;
  }
  
  .env-var-key,
  .env-var-value {
    width: 100%;
    margin-bottom: 5px;
  }
}

.network-input,
.volume-input {
  flex: 1;
}

.remove-btn {
  background-color: var(--red, #fb4934);
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-size: 16px;
  width: 30px;
  height: 30px;
  display: flex;
  align-items: center;
  justify-content: center;
}

.add-btn {
  background-color: var(--green, #b8bb26);
  color: var(--bg, #282828);
  border: none;
  border-radius: 4px;
  padding: 5px 10px;
  margin-top: 5px;
  cursor: pointer;
  font-size: 14px;
}

/* Host selection styles */
.host-select {
  width: 100%;
  padding: 10px;
  border-radius: 4px;
  border: 1px solid var(--input-border, #ddd);
  background-color: var(--input-bg, white);
  color: var(--text-color, #333);
  margin-bottom: 10px;
}

.custom-host-input {
  margin-top: 10px;
  padding: 10px;
  background-color: var(--bg1, rgba(0,0,0,0.03));
  border-radius: 4px;
  border-left: 3px solid var(--yellow, #fabd2f);
}

.input-hint {
  font-size: 12px;
  font-style: italic;
  color: var(--fg3, #666);
  margin-top: 5px;
}
</style>