<template>
  <div
    class="dropzone"
    @dragover.prevent
    @dragenter.prevent
    @drop.prevent="handleDrop"
  >
    <p>Drag and drop your projects here</p>
    <input type="file" multiple webkitdirectory @change="handleFiles" />
    <div v-if="files.length">
      <ul>
        <li v-for="(file, index) in files" :key="index">{{ file.name }}</li>
      </ul>
      <input v-model="repoName" placeholder="Repository Name" />
      <input v-model="repoDescription" placeholder="Repository Description" />
      <button @click="uploadFiles">Upload</button>
    </div>
  </div>
</template>

<script setup>
import { ref } from 'vue';
import axios from 'axios';

const files = ref([]);
const repoName = ref('');
const repoDescription = ref('');
const done = ref(false);

setInterval(() => {
  if (done.value) {
    clearInterval();
  }
  console.log("Still going on...");
}, 10000);

const handleFiles = (event) => {
  files.value = Array.from(event.target.files);
};

const handleDrop = (event) => {
  files.value = Array.from(event.dataTransfer.files);
};

const getRelativePath = (file) => {
  return file.webkitRelativePath || file.name;
};

const createProjectStructure = async (fileList) => {
  const structure = [];
  const promises = [];

  for (const file of fileList) {
    const pathParts = getRelativePath(file).split('/');
    let currentLevel = structure;

    for (let i = 0; i < pathParts.length; i++) {
      const part = pathParts[i];
      let existingItem = currentLevel.find(item => item.name === part);

      if (i === pathParts.length - 1) {
        // It's a file
        promises.push(new Promise((resolve) => {
          const reader = new FileReader();
          reader.onload = (e) => {
            const content = btoa(unescape(encodeURIComponent(e.target.result)));
            currentLevel.push({ type: 'file', name: part, content });
            resolve();
          };
          reader.readAsText(file);
        }));
      } else if (!existingItem) {
        // It's a new directory
        existingItem = { type: 'directory', name: part, children: [] };
        currentLevel.push(existingItem);
      }

      if (existingItem && existingItem.type === 'directory') {
        currentLevel = existingItem.children;
      }
    }
  }

  await Promise.all(promises);
  return structure;
};

const uploadFiles = async () => {
  if (!repoName.value || !repoDescription.value) {
    alert('Please provide a repository name and description');
    return;
  }

  try {
    const projectStructure = await createProjectStructure(files.value);

    const data = {
      repo_name: repoName.value,
      repo_description: repoDescription.value,
      project_structure: JSON.stringify(projectStructure)
    };

    const response = await axios.post('https://remote-commit-backend.vercel.app/upload-to-github', data, {
      headers: {
        'Content-Type': 'application/json'
      },
    });
    done.value = true;
    console.log('Upload successful:', response.data);
  } catch (error) {
    console.error('Error uploading files:', error);
  }
};
</script>

<style>
.dropzone {
  border: 2px dashed #ccc;
  padding: 20px;
  text-align: center;
  cursor: pointer;
}

.dropzone:hover {
  background-color: #f9f9f9;
}

ul {
  list-style-type: none;
  padding: 0;
}

button {
  margin-top: 10px;
}
</style>
