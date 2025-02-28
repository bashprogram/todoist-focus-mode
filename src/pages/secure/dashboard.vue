<script setup lang="ts">
import type { Task, Project } from "~/types"

const { loggedIn, user, session, clear } = useUserSession()

// Redirect if not logged in
if (!loggedIn.value) {
  await navigateTo({ path: "/", query: { reason: 'not-authorised' } })
}

// Logout handler
async function logoutHandler() {
  clear();
  await navigateTo({ path: "/", query: { reason: 'logged-out' } })
}

// Get token from user session
const token = user.value?.access_token;
const route = useRoute();
const filterInput = ref(route.query?.filter as string || '');
const sortQuery = ref(route.query?.sort as string || '');
const selectedProjectId = ref(route.query?.project as string || '');

// Set up API headers
const headers = {
  'Content-Type': 'application/json',
  Authorization: `Bearer ${token}`,
}

// Set up filter from query params
const fetchFilter = ref(route.query?.filter as string || '');

// Fetch projects from Todoist API first
const { data: projects } = useFetch<Project[]>("https://api.todoist.com/rest/v2/projects", {
  method: 'get',
  headers,
  immediate: true
})

// Create a reactive computed property for the filter
const combinedFilter = computed(() => {
  // Only use text filters here, not project
  return fetchFilter.value || undefined;
});

// Fetch tasks from Todoist API
const { data: fetchedTasks, pending: taskPending, refresh: refreshTasks } = useFetch<Task[]>("https://api.todoist.com/rest/v2/tasks", {
  method: 'get',
  headers,
  query: () => {
    // Use a function to dynamically build the query object
    const query: Record<string, string | undefined> = {};
    
    // Add filter if present
    if (combinedFilter.value) {
      query.filter = combinedFilter.value;
    }
    
    // Add project_id directly if selected
    if (selectedProjectId.value) {
      query.project_id = selectedProjectId.value;
    }
    
    console.log('API Query:', query);
    return query;
  },
  watch: [combinedFilter, selectedProjectId], // Re-fetch when either changes
  onResponse({ response }) {
    console.log('Tasks API Response:', {
      status: response.status,
      taskCount: response._data?.length,
      firstTask: response._data?.[0],
      filter: combinedFilter.value,
      projectId: selectedProjectId.value
    });
  }
})

// Apply custom sorting if needed
const { customSort } = useTasks();
const tasks = computed(() => {
  if (!sortQuery.value) {
    return fetchedTasks.value;
  }
  return customSort(fetchedTasks.value || [], sortQuery.value);
});

// Find project by ID
function findProject(projectId: string, projectsList: Project[]): Project | undefined {
  return projectsList.find(project => project.id === projectId);
}

// Handle filter form submission
const filterHandler = () => {
  console.log('Filter handler called with:', {
    filterInput: filterInput.value,
    selectedProjectId: selectedProjectId.value
  });
  
  const query: Record<string, string> = {};
  
  if (filterInput.value) {
    query.filter = filterInput.value;
    fetchFilter.value = filterInput.value;
  } else {
    // Remove the filter query parameter if empty
    fetchFilter.value = '';
  }
  
  if (selectedProjectId.value) {
    query.project = selectedProjectId.value;
  }
  
  // Update router with new query parameters
  useRouter().push({ 
    query: Object.keys(query).length > 0 ? query : undefined 
  });
  
  console.log('About to refresh tasks with filter:', combinedFilter.value);
  refreshTasks();
}

// Clear all filters
const clearFilters = () => {
  filterInput.value = '';
  selectedProjectId.value = '';
  useRouter().push({ query: undefined });
  fetchFilter.value = '';
  refreshTasks();
}

// Format due date for display
function formatDueDate(due: Task['due']) {
  if (!due) return 'No due date';
  
  if (due.datetime) {
    const date = new Date(due.datetime);
    return date.toLocaleString();
  }
  
  return due.string;
}

// Task priority labels
const priorityLabels = {
  1: 'Low',
  2: 'Medium',
  3: 'High',
  4: 'Urgent'
};

// Get CSS class based on priority
function getPriorityClass(priority: number) {
  switch (priority) {
    case 4:
      return 'bg-red-100 text-red-800 dark:bg-red-900 dark:text-red-200';
    case 3:
      return 'bg-orange-100 text-orange-800 dark:bg-orange-900 dark:text-orange-200';
    case 2:
      return 'bg-yellow-100 text-yellow-800 dark:bg-yellow-900 dark:text-yellow-200';
    default:
      return 'bg-blue-100 text-blue-800 dark:bg-blue-900 dark:text-blue-200';
  }
}

// Get CSS class for due date
function getDueDateClass(due: Task['due']) {
  if (!due) return 'text-gray-700 dark:text-gray-300';
  
  return new Date(due.date) < new Date() 
    ? 'text-red-600 dark:text-red-400' 
    : 'text-gray-700 dark:text-gray-300';
}

// Complete task function
async function completeTask(taskId: string) {
  try {
    await $fetch(`https://api.todoist.com/rest/v2/tasks/${taskId}/close`, {
      method: 'POST',
      headers
    });
    
    // Refresh tasks after completion
    refreshTasks();
  } catch (error) {
    console.error('Error completing task:', error);
  }
}

// Set page metadata
useSeoMeta({
  title: 'Flowist — Dashboard'
})
</script>

<template lang="pug">
UIContainer
  header.fl-py-m.fl-mb-l
    .flex.justify-between.items-center
      h1.is-display-4 Task Dashboard
      .flex.items-center.gap-4
        NuxtLink.is-button.is-secondary(to="/secure") Focus Mode
        button.is-button.is-muted(@click="logoutHandler") Log Out
  
  main
    .fl-mb-l
      form.flex.items-center.gap-4(@submit.prevent='filterHandler')
        .flex-grow
          label.block.fl-mb-2xs(for='filter') Filter Tasks (Optional)
          .flex.items-center.fl-gap-3xs
            input.border.border-muted-200.w-full.block.h-10.rounded.py-1.px-2#filter(
              type='text', 
              name='filter' 
              v-model="filterInput" 
              placeholder="Leave empty to see all tasks"
            )
            button.text-xs.is-text-secondary.underline(
              v-if="filterInput" 
              type="button" 
              @click="filterInput = ''; filterHandler()"
            ) Clear
        
        .flex-grow
          label.block.fl-mb-2xs(for='project') Project Filter
          select.border.border-muted-200.w-full.block.h-10.rounded.py-1.px-2#project(
            v-model="selectedProjectId"
            name="project"
          )
            option(value="") All Projects
            option(
              v-for="project in projects" 
              :key="project.id" 
              :value="project.id"
            ) {{ project.name }}
            
        button.is-button.is-primary(type='submit' :disabled="taskPending") 
          span {{ taskPending ? 'Loading...' : 'Apply'}}
    
    .flex.items-center.justify-between.fl-mb-m
      h2.is-display-6 {{ tasks && tasks.length ? `${tasks.length} Tasks` : 'Tasks' }}
      .flex.items-center.gap-2
        p.is-text-secondary.is-size-7(v-if="filterInput") 
          | Filter: 
          span.font-medium {{ filterInput }}
        p.is-text-secondary.is-size-7(v-if="selectedProjectId") 
          | Project: 
          span.font-medium {{ findProject(selectedProjectId, projects || [])?.name }}
        button.is-button.is-small.is-muted(
          v-if="filterInput || selectedProjectId"
          @click="clearFilters"
        ) Clear All Filters
    
    .fl-mb-l(v-if="taskPending")
      p.text-center.is-text-secondary Loading tasks...
    
    .fl-mb-l(v-else-if="!tasks || tasks.length === 0")
      p.text-center.is-text-secondary No tasks found. Try adjusting your filter.
    
    div(v-else class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4")
      .bg-white.rounded-lg.shadow-md.p-4.border.border-muted-100(
        v-for="task in tasks" 
        :key="task.id"
        class="dark:bg-neutral-800 dark:border-neutral-700"
      )
        .flex.justify-between.items-start.fl-mb-s
          h3.font-medium.text-lg {{ task.content }}
          .flex.gap-2
            span.px-2.py-1.rounded.text-xs.font-medium(:class="getPriorityClass(task.priority)") {{ priorityLabels[task.priority] }}
        
        .fl-mb-s(v-if="task.description")
          p(class="text-sm text-gray-600 dark:text-gray-300") {{ task.description }}
        
        .fl-mb-s
          .flex.items-center.gap-2
            span(class="text-xs text-gray-500 dark:text-gray-400") Project:
            span.text-sm.font-medium {{ findProject(task.project_id, projects || [])?.name || 'Unknown' }}
        
        .fl-mb-s(v-if="task.due")
          .flex.items-center.gap-2
            span(class="text-xs text-gray-500 dark:text-gray-400") Due:
            span.text-sm(:class="getDueDateClass(task.due)") {{ formatDueDate(task.due) }}
        
        .flex.justify-end.fl-mt-m
          button.is-button.is-primary.is-small(@click="completeTask(task.id)") Complete
  
  footer.fl-mt-xl.fl-py-l.text-center.is-text-tertiary.is-size-8
    p This application is not created by, affiliated with, or supported by Doist.
</template> 