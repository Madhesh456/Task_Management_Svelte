<script lang="ts">
  import { writable, derived } from 'svelte/store';
  import { onMount } from 'svelte';

  // --- Type Definitions ---
  type TaskStatus = 'Pending' | 'In-Progress' | 'Completed';

  interface Task {
    id: string;
    title: string;
    description: string;
    status: TaskStatus;
    createdAt: string;
  }

  // --- Initial State (Mock API Data) ---
  const initialTasks: Task[] = [
    {
      id: '1',
      title: 'Design Dashboard UI with a focus on a clean and modern user experience',
      description: 'Create a clean and modern design for the task dashboard, including wireframes and mockups that are responsive on all devices. The design should be simple, intuitive, and easy to navigate for users of all skill levels. Pay special attention to color palettes and typography to create a visually appealing interface. It is important to create a comprehensive design system that will allow for easy scalability in the future. The design should also include mockups for both light and dark mode.',
      status: 'In-Progress',
      createdAt: new Date(Date.now() - 86400000).toLocaleDateString(),
    },
    {
      id: '2',
      title: 'Implement Task Creation and management functionality',
      description: 'Develop the form and logic for adding new tasks to the list. This includes input validation, status selection, and a confirmation dialog. The implementation should be robust and handle potential errors gracefully. It is a good idea to consider both server-side and client-side validation to ensure data integrity. The task creation form should also be accessible to users with disabilities, following WCAG guidelines.',
      status: 'Completed',
      createdAt: new Date(Date.now() - 172800000).toLocaleDateString(),
    },
    {
      id: '3',
      title: 'Set up Project Environment and configure development tools',
      description: 'Initialize the Svelte, TypeScript, and Tailwind CSS project using Vite or SvelteKit. Configure ESLint and Prettier for code quality and consistency. Set up a component-based architecture for reusability and maintainability across the entire application. It is important to document all the setup steps clearly in the project README.',
      status: 'Completed',
      createdAt: new Date(Date.now() - 259200000).toLocaleDateString(),
    },
    {
      id: '4',
      title: 'Add a comprehensive Search and Filter feature to the task list',
      description: 'Implement functionality to search by title and description and filter by task status. This feature should be fast and responsive, providing immediate feedback to the user as they type or change the filter. The search algorithm should be efficient and case-insensitive. Consider adding a debounce function to the search input to prevent excessive re-renders.',
      status: 'Pending',
      createdAt: new Date().toLocaleDateString(),
    },
  ];

  // --- State Management ---
  const tasks = writable<Task[]>([]);
  const searchTerm = writable('');
  const filterStatus = writable<TaskStatus | 'All'>('All');
  const isDarkMode = writable(false);
  let isContentLoading = true;

  // --- Component State ---
  let isEditModalOpen = false;
  let isDeleteModalOpen = false;
  let isAddModalOpen = false;
  let isLoading = false;
  let currentTask: Task | null = null;
  let newTaskTitle = '';
  let newTaskDescription = '';
  let newTaskStatus: TaskStatus = 'Pending';
  let confirmMessage = '';
  let pendingTaskToDelete: Task | null = null;
  const MAX_CHARACTERS = 500;

  // --- Truncation and Utility Functions ---
  function truncateText(text: string, maxWords: number): string {
    if (!text) return '';
    const words = text.split(' ');
    if (words.length > maxWords) {
      return words.slice(0, maxWords).join(' ') + '...';
    }
    return text;
  }

  // --- Reactive Filtering and Searching ---
  const filteredTasks = derived(
    [tasks, searchTerm, filterStatus],
    ([$tasks, $searchTerm, $filterStatus]) => {
      const filteredByStatus = $filterStatus === 'All'
        ? $tasks
        : $tasks.filter(task => task.status === $filterStatus);

      return $searchTerm
        ? filteredByStatus.filter(task =>
            task.title.toLowerCase().includes($searchTerm.toLowerCase()) ||
            task.description.toLowerCase().includes($searchTerm.toLowerCase())
          )
        : filteredByStatus;
    }
  );

  // --- Lifecycle Hooks ---
  onMount(() => {
    // Set initial dark mode based on system preference
    const mediaQuery = window.matchMedia('(prefers-color-scheme: dark)');
    $isDarkMode = mediaQuery.matches;

    // Optional: Add a listener for system theme changes
    mediaQuery.addEventListener('change', (e) => {
      $isDarkMode = e.matches;
    });

    // Simulate lazy loading of content
    setTimeout(() => {
      tasks.set(initialTasks);
      isContentLoading = false;
    }, 1500); // Simulate network delay
  });

  // --- Event Handlers ---
  async function addTask() {
    if (!newTaskTitle.trim() || newTaskDescription.length > MAX_CHARACTERS) return;

    isLoading = true;
    await new Promise(resolve => setTimeout(resolve, 1500)); // Simulate API call delay

    const newTask: Task = {
      id: crypto.randomUUID(),
      title: newTaskTitle,
      description: newTaskDescription,
      status: newTaskStatus,
      createdAt: new Date().toLocaleDateString(),
    };
    tasks.update(currentTasks => [...currentTasks, newTask]);
    
    isLoading = false;
    isAddModalOpen = false;
    resetNewTaskForm();
  }

  function resetNewTaskForm() {
    newTaskTitle = '';
    newTaskDescription = '';
    newTaskStatus = 'Pending';
  }

  function handleCancelAddTask() {
    resetNewTaskForm();
    isAddModalOpen = false;
  }

  function openEditModal(task: Task) {
    currentTask = { ...task };
    isEditModalOpen = true;
  }

  function saveEditedTask() {
    if (currentTask && currentTask.description.length <= MAX_CHARACTERS) {
      tasks.update(currentTasks =>
        currentTasks.map(task =>
          task.id === currentTask?.id ? currentTask : task
        )
      );
      isEditModalOpen = false;
    }
  }

  function openDeleteModal(task: Task) {
    pendingTaskToDelete = task;
    confirmMessage = `Are you sure you want to delete the task titled "${task.title}"?`;
    isDeleteModalOpen = true;
  }

  function deleteTask() {
    if (pendingTaskToDelete) {
      tasks.update(currentTasks =>
        currentTasks.filter(task => task.id !== pendingTaskToDelete?.id)
      );
    }
    closeDeleteModal();
  }

  function closeDeleteModal() {
    isDeleteModalOpen = false;
    pendingTaskToDelete = null;
    confirmMessage = '';
  }

  function toggleDarkMode() {
    isDarkMode.update(dark => !dark);
  }

  // --- Accessibility and Status Styling ---
  function getStatusStyle(status: TaskStatus) {
    switch (status) {
      case 'Pending':
        return 'bg-blue-100 text-blue-800 dark:bg-blue-900 dark:text-blue-200';
      case 'In-Progress':
        return 'bg-yellow-100 text-yellow-800 dark:bg-yellow-900 dark:text-yellow-200';
      case 'Completed':
        return 'bg-green-100 text-green-800 dark:bg-green-900 dark:text-green-200';
    }
  }

  function getStatusIcon(status: TaskStatus) {
    switch (status) {
      case 'Pending':
        return '⏳';
      case 'In-Progress':
        return '🚧';
      case 'Completed':
        return '✅';
    }
  }
</script>

<svelte:head>
  <!-- Basic SEO Meta Tags -->
  <meta name="description" content="A modern and responsive task management dashboard built with Svelte and Tailwind CSS. Create, edit, and filter your tasks efficiently.">
  <meta name="keywords" content="Svelte, Tailwind CSS, Task Management, Dashboard, Frontend, Web Development">
  <meta name="author" content="Your Name">

  <!-- Load Inter font from Google Fonts -->
  <link rel="stylesheet" href="https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700&display=swap">
  <!-- Load Tailwind CSS from CDN -->
  <script src="https://cdn.tailwindcss.com"></script>
  <script>
    if (typeof tailwind !== 'undefined') {
      tailwind.config = {
        darkMode: 'class',
        theme: {
          extend: {
            fontFamily: {
              sans: ['Inter', 'sans-serif'],
            },
          },
        },
      };
    }
  </script>
</svelte:head>

<!-- --- Main Application Layout --- -->
<main class="font-sans min-h-screen transition-colors duration-300" class:dark={$isDarkMode} style="background-color: {$isDarkMode ? '#1a202c' : '#f7fafc'};">

  {#if isContentLoading}
    <!-- Full-page Skeleton Loader -->
    <header class="sticky top-0 z-10 w-full bg-gray-100 dark:bg-gray-800 shadow-xl transition-colors duration-300 animate-pulse">
      <div class="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8 py-4 flex flex-col md:flex-row items-center justify-between gap-4">
        <div class="h-10 w-48 bg-gray-300 dark:bg-gray-700 rounded-lg"></div>
        <div class="flex flex-col sm:flex-row items-center space-y-4 sm:space-y-0 sm:space-x-4 w-full md:w-auto">
          <div class="w-full md:w-64 h-10 bg-gray-300 dark:bg-gray-700 rounded-lg"></div>
          <div class="w-full h-10 bg-gray-300 dark:bg-gray-700 rounded-full"></div>
          <div class="w-24 h-6 bg-gray-300 dark:bg-gray-700 rounded-full"></div>
        </div>
      </div>
    </header>

    <div class="max-w-7xl mx-auto p-4 sm:p-8 pt-6 sm:pt-10">
      <div class="mt-6 flex flex-wrap gap-2 justify-center animate-pulse">
        <div class="w-20 h-8 rounded-full bg-gray-300 dark:bg-gray-700"></div>
        <div class="w-24 h-8 rounded-full bg-gray-300 dark:bg-gray-700"></div>
        <div class="w-28 h-8 rounded-full bg-gray-300 dark:bg-gray-700"></div>
        <div class="w-28 h-8 rounded-full bg-gray-300 dark:bg-gray-700"></div>
      </div>
      <div class="mt-8 grid gap-6 sm:grid-cols-1 lg:grid-cols-2">
        {#each Array(4) as _, i}
          <div class="group bg-gray-200 dark:bg-gray-700 rounded-xl p-5 shadow-sm border border-gray-300 dark:border-gray-600 flex flex-col justify-between animate-pulse">
            <div>
              <div class="flex flex-col sm:flex-row sm:justify-between sm:items-center mb-2">
                <div class="h-6 w-3/4 bg-gray-300 dark:bg-gray-600 rounded"></div>
                <div class="h-6 w-16 bg-gray-300 dark:bg-gray-600 rounded-full mt-2 sm:mt-0"></div>
              </div>
              <div class="space-y-2">
                <div class="h-4 bg-gray-300 dark:bg-gray-600 rounded"></div>
                <div class="h-4 w-5/6 bg-gray-300 dark:bg-gray-600 rounded"></div>
              </div>
            </div>
            <div class="mt-4 pt-4 border-t border-gray-300 dark:border-gray-600 flex items-center justify-between text-sm">
              <div class="h-4 w-1/3 bg-gray-300 dark:bg-gray-600 rounded"></div>
              <div class="flex space-x-2">
                <div class="h-8 w-12 bg-gray-300 dark:bg-gray-600 rounded-md"></div>
                <div class="h-8 w-12 bg-gray-300 dark:bg-gray-600 rounded-md"></div>
              </div>
            </div>
          </div>
        {/each}
      </div>
    </div>
  {:else}
    <!-- Header and Controls (Sticky) -->
    <header class="sticky top-0 z-10 w-full bg-white dark:bg-gray-800 shadow-xl transition-colors duration-300">
      <div class="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8 py-4 flex flex-col md:flex-row items-center justify-between gap-4">
        <h1 class="text-3xl sm:text-4xl font-bold text-gray-900 dark:text-white">Task Dashboard</h1>
        <div class="flex flex-col sm:flex-row items-center space-y-4 sm:space-y-0 sm:space-x-4 w-full md:w-auto">
          <label for="search" class="sr-only">Search</label>
          <input
            id="search"
            type="text"
            bind:value={$searchTerm}
            placeholder="Search tasks..."
            class="w-full md:w-64 px-4 py-2 text-sm rounded-lg border border-gray-300 dark:border-gray-600 bg-gray-50 dark:bg-gray-700 dark:text-gray-200 placeholder-gray-500 dark:placeholder-gray-400 focus:outline-none focus:ring-2 focus:ring-blue-500 transition-colors duration-200"
          >
          <!-- Refreshed "Add New Task" button -->
          <button
            on:click={() => isAddModalOpen = true}
            class="w-full px-6 py-3 text-sm font-medium text-white rounded-full shadow-lg hover:shadow-xl active:scale-95 transform transition-all duration-300 focus:outline-none focus:ring-4 focus:ring-purple-300 bg-gradient-to-r from-blue-500 to-purple-600 flex items-center justify-center gap-2"
          >
            <svg xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke-width="2.5" stroke="currentColor" class="w-5 h-5">
              <path stroke-linecap="round" stroke-linejoin="round" d="M12 4.5v15m7.5-7.5h-15" />
            </svg>
            Add New Task
          </button>
          <!-- Themed dark mode toggle button -->
          <label for="dark-mode-toggle" class="relative inline-flex items-center cursor-pointer flex-shrink-0">
            <input type="checkbox" id="dark-mode-toggle" class="sr-only peer" checked={$isDarkMode} on:change={toggleDarkMode}>
            <div class="w-11 h-6 bg-gray-200 peer-focus:outline-none peer-focus:ring-4 peer-focus:ring-blue-300 dark:peer-focus:ring-blue-800 rounded-full peer dark:bg-gray-700 peer-checked:after:translate-x-full peer-checked:after:border-white after:content-[''] after:absolute after:top-[2px] after:left-[2px] after:bg-white after:border-gray-300 after:border after:rounded-full after:h-5 after:w-5 after:transition-all dark:border-gray-600 peer-checked:bg-blue-600"></div>
            <span class="ml-3 text-sm font-medium text-gray-900 dark:text-gray-300">Dark Mode</span>
          </label>
        </div>
      </div>
    </header>

    <!-- Main content area -->
    <div class="max-w-7xl mx-auto p-4 sm:p-8 pt-6 sm:pt-10">
      <!-- Filter Buttons -->
      <div class="mt-6 flex flex-wrap gap-2 justify-center">
        {#each ['All', 'Pending', 'In-Progress', 'Completed'] as status}
          <button
            on:click={() => $filterStatus = status as TaskStatus | 'All'}
            class="px-4 py-2 text-sm font-medium rounded-full transition-colors duration-200"
            class:bg-blue-600={status === $filterStatus}
            class:text-white={status === $filterStatus}
            class:bg-gray-200={status !== $filterStatus}
            class:text-gray-700={status !== $filterStatus}
            class:dark:bg-gray-700={status !== $filterStatus && $isDarkMode}
            class:dark:text-gray-200={status !== $filterStatus && $isDarkMode}
            aria-pressed={status === $filterStatus}
          >
            {status}
          </button>
        {/each}
      </div>

      <!-- Task List -->
      <div class="mt-8 grid gap-6 sm:grid-cols-1 lg:grid-cols-2">
        {#if $filteredTasks.length === 0}
          <p class="text-center text-gray-500 dark:text-gray-400 col-span-full py-10">No tasks found matching your criteria.</p>
        {:else}
          {#each $filteredTasks as task (task.id)}
            <div 
              class="group bg-gray-50 dark:bg-gray-700 rounded-xl p-5 shadow-sm border border-gray-200 dark:border-gray-600 flex flex-col justify-between transition-transform duration-200 hover:scale-[1.01] transform-gpu"
              title={task.description}
            >
              <div>
                <div class="flex flex-col sm:flex-row sm:justify-between sm:items-center mb-2">
                  <h3 class="text-lg font-semibold text-gray-900 dark:text-white sm:mr-4 sm:max-w-[70%] sm:truncate">{task.title}</h3>
                  <span class="px-3 py-1 text-xs font-semibold rounded-full mt-2 sm:mt-0 flex-shrink-0 {getStatusStyle(task.status)}">
                    {getStatusIcon(task.status)} {task.status}
                  </span>
                </div>
                <p class="text-sm text-gray-600 dark:text-gray-400">{truncateText(task.description, 100)}</p>
              </div>
              <div class="mt-4 pt-4 border-t border-gray-200 dark:border-gray-600 flex items-center justify-between text-sm text-gray-500 dark:text-gray-400">
                <span>Created on: {task.createdAt}</span>
                <div class="flex space-x-2">
                  <button
                    on:click={() => openEditModal(task)}
                    class="px-3 py-1 text-xs font-medium bg-blue-500 text-white rounded-md shadow-sm hover:bg-blue-600 focus:outline-none focus:ring-2 focus:ring-blue-500 transition-colors duration-200"
                    aria-label="Edit task"
                  >
                    Edit
                  </button>
                  <button
                    on:click={() => openDeleteModal(task)}
                    class="px-3 py-1 text-xs font-medium bg-red-500 text-white rounded-md shadow-sm hover:bg-red-600 focus:outline-none focus:ring-2 focus:ring-red-500 transition-colors duration-200"
                    aria-label="Delete task"
                  >
                    Delete
                  </button>
                </div>
              </div>
            </div>
          {/each}
        {/if}
      </div>
    </div>
  {/if}

  <!-- --- Modals --- -->
  {#if isAddModalOpen}
    <!-- Add Task Modal -->
    <div
      class="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center p-4 z-50"
      aria-modal="true"
      role="dialog"
      aria-labelledby="add-modal-title"
    >
      <div class="bg-white dark:bg-gray-800 rounded-xl shadow-2xl p-6 w-full max-w-lg lg:max-w-xl transition-transform duration-300 scale-100">
        <h2 id="add-modal-title" class="text-xl font-bold text-gray-900 dark:text-white mb-4">Create New Task</h2>
        <div class="p-6 bg-gray-50 dark:bg-gray-700 rounded-xl shadow-inner border border-gray-200 dark:border-gray-600">
          <form on:submit|preventDefault={addTask} class="grid gap-6">
            <label for="new-task-title" class="block">
              <span class="text-sm font-medium text-gray-700 dark:text-gray-300">Title <span class="text-red-500">*</span></span>
              <input
                id="new-task-title"
                type="text"
                bind:value={newTaskTitle}
                required
                class="mt-1 w-full px-4 py-2 rounded-lg border border-gray-300 dark:border-gray-600 bg-white dark:bg-gray-800 dark:text-gray-200 focus:outline-none focus:ring-2 focus:ring-blue-500 transition-colors duration-200"
              >
            </label>
            <label for="new-task-description" class="block">
              <span class="text-sm font-medium text-gray-700 dark:text-gray-300" class:text-red-500={newTaskDescription.length > MAX_CHARACTERS}>Description ({newTaskDescription.length}/{MAX_CHARACTERS} characters)</span>
              <textarea
                id="new-task-description"
                bind:value={newTaskDescription}
                rows="3"
                maxlength={MAX_CHARACTERS}
                class="mt-1 w-full px-4 py-2 rounded-lg border dark:border-gray-600 bg-white dark:bg-gray-800 dark:text-gray-200 focus:outline-none focus:ring-2 focus:ring-blue-500 transition-colors duration-200 max-h-36 overflow-y-auto"
                class:border-gray-300={newTaskDescription.length <= MAX_CHARACTERS}
                class:border-red-500={newTaskDescription.length > MAX_CHARACTERS}
              ></textarea>
            </label>
            <label for="new-task-status" class="block">
              <span class="text-sm font-medium text-gray-700 dark:text-gray-300">Status</span>
              <select
                id="new-task-status"
                bind:value={newTaskStatus}
                class="mt-1 w-full px-4 py-2 rounded-lg border border-gray-300 dark:border-gray-600 bg-white dark:bg-gray-800 dark:text-gray-200 focus:outline-none focus:ring-2 focus:ring-blue-500 transition-colors duration-200"
              >
                <option value="Pending">Pending</option>
                <option value="In-Progress">In-Progress</option>
                <option value="Completed">Completed</option>
              </select>
            </label>
            <div class="flex justify-end space-x-2 mt-4">
              <button
                type="button"
                on:click={handleCancelAddTask}
                class="px-6 py-2 text-sm font-medium rounded-lg text-gray-600 dark:text-gray-300 bg-transparent hover:bg-gray-200 dark:hover:bg-gray-700 focus:outline-none focus:ring-2 focus:ring-gray-400 transition-all duration-200"
              >
                Cancel
              </button>
              <button
                type="submit"
                class="px-6 py-2 text-sm font-medium text-white bg-blue-600 rounded-lg shadow-lg hover:bg-blue-700 focus:outline-none focus:ring-2 focus:ring-blue-500 transition-all duration-200"
                disabled={isLoading || newTaskDescription.length > MAX_CHARACTERS}
              >
                {#if isLoading}
                  Adding...
                {:else}
                  Add Task
                {/if}
              </button>
            </div>
          </form>
        </div>
      </div>
    </div>
  {/if}
  
  {#if isEditModalOpen}
    <!-- Edit Modal -->
    <div
      class="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center p-4 z-50"
      aria-modal="true"
      role="dialog"
      aria-labelledby="edit-modal-title"
    >
      <div class="bg-white dark:bg-gray-800 rounded-xl shadow-2xl p-6 w-full max-w-lg lg:max-w-xl transition-transform duration-300 scale-100">
        <h2 id="edit-modal-title" class="text-xl font-bold text-gray-900 dark:text-white mb-4">Edit Task</h2>
        {#if currentTask}
          <form on:submit|preventDefault={saveEditedTask} class="space-y-4">
            <label for="edit-title" class="block">
              <span class="text-sm font-medium text-gray-700 dark:text-gray-300">Title <span class="text-red-500">*</span></span>
              <input
                id="edit-title"
                type="text"
                bind:value={currentTask.title}
                required
                class="mt-1 w-full px-4 py-2 rounded-lg border border-gray-300 dark:border-gray-600 bg-white dark:bg-gray-800 dark:text-gray-200 focus:outline-none focus:ring-2 focus:ring-blue-500 transition-colors duration-200"
              >
            </label>
            <label for="edit-description" class="block">
              <span class="text-sm font-medium text-gray-700 dark:text-gray-300" class:text-red-500={currentTask.description.length > MAX_CHARACTERS}>Description ({currentTask.description.length}/{MAX_CHARACTERS} characters)</span>
              <textarea
                id="edit-description"
                bind:value={currentTask.description}
                rows="3"
                maxlength={MAX_CHARACTERS}
                class="mt-1 w-full px-4 py-2 rounded-lg border dark:border-gray-600 bg-white dark:bg-gray-800 dark:text-gray-200 focus:outline-none focus:ring-2 focus:ring-blue-500 transition-colors duration-200 max-h-36 overflow-y-auto"
                class:border-gray-300={currentTask.description.length <= MAX_CHARACTERS}
                class:border-red-500={currentTask.description.length > MAX_CHARACTERS}
              ></textarea>
            </label>
            <label for="edit-status" class="block">
              <span class="text-sm font-medium text-gray-700 dark:text-gray-300">Status</span>
              <select
                id="edit-status"
                bind:value={currentTask.status}
                class="mt-1 w-full px-4 py-2 rounded-lg border border-gray-300 dark:border-gray-600 bg-white dark:bg-gray-800 dark:text-gray-200 focus:outline-none focus:ring-2 focus:ring-blue-500 transition-colors duration-200"
              >
                <option value="Pending">Pending</option>
                <option value="In-Progress">In-Progress</option>
                <option value="Completed">Completed</option>
              </select>
            </label>
            <div class="flex justify-end space-x-2 mt-4">
              <button
                type="button"
                on:click={() => isEditModalOpen = false}
                class="px-4 py-2 text-sm font-medium text-gray-700 dark:text-gray-300 bg-gray-200 dark:bg-gray-700 rounded-md hover:bg-gray-300 dark:hover:bg-gray-600 focus:outline-none focus:ring-2 focus:ring-gray-400 transition-colors duration-200"
              >
                Cancel
              </button>
              <button
                type="submit"
                class="px-4 py-2 text-sm font-medium bg-blue-600 text-white rounded-md hover:bg-blue-700 focus:outline-none focus:ring-2 focus:ring-blue-500 transition-colors duration-200"
                disabled={currentTask.description.length > MAX_CHARACTERS}
              >
                Save Changes
              </button>
            </div>
          </form>
        {/if}
      </div>
    </div>
  {/if}

  {#if isDeleteModalOpen}
    <!-- Delete Confirmation Modal -->
    <div
      class="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center p-4 z-50"
      aria-modal="true"
      role="dialog"
      aria-labelledby="delete-modal-title"
    >
      <div class="bg-white dark:bg-gray-800 rounded-xl shadow-2xl p-6 w-full max-w-sm lg:max-w-md transition-transform duration-300 scale-100">
        <h2 id="delete-modal-title" class="text-lg font-bold text-gray-900 dark:text-white mb-2">Confirm Deletion</h2>
        <p class="text-sm text-gray-600 dark:text-gray-400 mb-4">{confirmMessage}</p>
        <div class="flex justify-end space-x-2">
          <button
            type="button"
            on:click={closeDeleteModal}
            class="px-4 py-2 text-sm font-medium text-gray-700 dark:text-gray-300 bg-gray-200 dark:bg-gray-700 rounded-md hover:bg-gray-300 dark:hover:bg-gray-600 focus:outline-none focus:ring-2 focus:ring-gray-400 transition-colors duration-200"
          >
            Cancel
          </button>
          <button
            type="button"
            on:click={deleteTask}
            class="px-4 py-2 text-sm font-medium bg-red-600 text-white rounded-md hover:bg-red-700 focus:outline-none focus:ring-2 focus:ring-red-500 transition-colors duration-200"
          >
            Delete
          </button>
        </div>
      </div>
    </div>
  {/if}

  {#if isLoading}
    <!-- Loader Overlay -->
    <div class="fixed inset-0 bg-gray-900 bg-opacity-50 flex items-center justify-center z-50">
      <div class="animate-spin rounded-full h-16 w-16 border-t-2 border-b-2 border-blue-500"></div>
    </div>
  {/if}
</main>
