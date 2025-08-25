<template>
  <div class="bill-splitter-app">
<!-- Popup Tip Modal-->
    <div v-if="showTipPopup" class="modal modal-open">
      <div class="modal-box">
        <h3 class="font-bold text-lg">Save some time 💡</h3>
        <p class="py-4">If no one is selected in the <b>Split With</b> column, the item will be split with everyone.</p>
        <div class="modal-action flex justify-between items-center">
          <label class="label cursor-pointer">
            <input type="checkbox" v-model="dontShowAgain" class="checkbox" />
            <span class="label-text ml-2">Don't show this again</span>
          </label>
          <button @click="closePopup" class="btn btn-primary">Close</button>
        </div>
      </div>
    </div>

<!-- Title -->
    <h1 class="text-3xl font-bold text-center mb-6">SplitBetter</h1>

<!-- Scan Receipt - Section Header -->
    <div class="mb-4">
      <span class="text-3xl">Scan Receipt</span>
    </div>

    <div class="bg-base-200 rounded-xl py-4 mb-6">
      <!-- Item input field -->
      <div class="mb-4 flex justify-center">
        <form v-if="showManualItemInput" autocomplete="off" @submit.prevent="addMenuItem" class="w-auto">
          <input
            type="text"
            v-model="newMenuItemName"
            placeholder="Item name"
            class="input"
            autocomplete="off"
          />
          <button type="submit" style="display: none;">Add Item</button>
        </form>
      </div>

      <div class="flex justify-center items-center gap-4 mb-4">
        <!-- Manual input button -->
        <button @click="showManualItemInput = !showManualItemInput" class="btn btn-outline">Manually add items</button>
        <div v-if="uploadError" class="text-error">{{ uploadError }}</div>
        <div v-if="isUploading">Processing receipt... 🤖</div>

        <!-- Upload receipt button -->
        <label v-else class="btn btn-accent">
          <input type="file" @change="handleReceiptUpload" accept="image/*" class="hidden" />
          📤 Upload Receipt
        </label>
      </div>

      <!-- Progress bar cluster -->
      <div class="flex flex-col gap-4 justify-center items-center">
        <!-- Progress bar -->
        <div >
          <progress @click="animateProgressBar" ref="progressBar"  class="progress progress-success w-56" value="0" max="100"></progress>
        </div>
        <!-- Start and finish buttons -->
        <div class="flex gap-4" v-if="inDevMode">
          <button @click="startProgressBarAnimation" class="btn">Start</button>
          <button @click="finishProgressBarAnimation" class="btn btn-primary">Finish</button>
        </div>
      </div>
    </div>

<!-- People - Section Header -->
    <div class="mb-4" id="add-people-section">
      <span class="text-3xl">People</span>
    </div>

    <div class="bg-base-200 rounded-xl py-4 mb-6">
      <!-- Name input -->
      <div class="flex justify-center gap-4 mb-4">
        <form autocomplete="off" @submit.prevent="handleAddPerson" class="w-auto">
          <input
            type="text"
            v-model="newPersonName"
            placeholder="Add name to split with"
            id="person-input"
            class="input w-full"
            autocomplete="off"
          />
          <button type="submit" style="display: none;">Add Person</button>
        </form>
      </div>

      <!-- Person list -->
      <div id="person-list" class="flex flex-wrap gap-2 justify-center">
      <span v-if="persons.length === 0" class="label-text">No people added yet.</span>
      <strong v-else>Guests:</strong>
      <div
        v-for="person in persons"
        :key="person"
        class="badge badge-lg badge-outline"
      >
        {{ person }}
      </div>
    </div>
    </div>

<!-- Items - Section Header -->
    <div class="mb-4">
      <span class="text-3xl">Items</span>
    </div>

    <div v-if="billItems.length === 0" class="text-center italic label-text">Upload a receipt to get started!</div>
    <div v-else>
      <div class="flex flex-col gap-4">
        <div v-for="item in billItems" :key="item.id" class="card card-bordered bg-base-200">
          <div class="card-body p-4 relative">
            <div class="flex flex-col gap-2">
              <div class="flex flex-nowrap gap-2 w-full items-center">
                <input type="text" v-model="item.name" data-testid="item-name" placeholder="Item" class="input"/>
                <input type="tel" :value="item.price" @input="event => formatPrice(item, event)" placeholder="0.00" class="input w-1/3 mr-8" />
              </div>
              <div class="mt-2">
                <div class="flex flex-wrap gap-1">
                  <button
                    v-for="person in persons"
                    :key="person"
                    @click="togglePersonForItem(item, person)"
                    :class="item.splitWith.includes(person) ? 'btn-outline' : 'btn-soft'"
                    class="btn btn-sm"
                    :title="person"
                  >
                    {{ person }}
                  </button>
                </div>
              </div>
            </div>
            <button @click="removeItem(item.id)" class="btn btn-sm btn-ghost btn-circle absolute top-2 right-2" title="Remove row">
              <svg xmlns="http://www.w3.org/2000/svg" class="h-4 w-4" fill="none" viewBox="0 0 24 24" stroke="currentColor"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M6 18L18 6M6 6l12 12" /></svg>
            </button>
          </div>
        </div>
      </div>
    </div>
    <button @click="addItemRow()" class="btn btn-accent w-full mt-4">+ Add Item Row</button>


<!-- Split Bill - Section Header -->
<div class="flex mb-4 mt-6">
  <p class="text-3xl mr-6">Summary</p>
</div>

<!-- Subtotal and Total section -->
    <div class="text-center my-4">
      <h3 class="text-xl font-semibold">Subtotal: ${{ subtotal.toFixed(2) }}</h3>
      <h3 class="text-xl font-semibold">Items: {{ billItems.length }}</h3>
    </div>

    <div class="form-control text-center">
      <span class=" mb-4">Total, after tip & tax:</span>
      <input type="tel" class="input base-content" :value="totalAmount" @input="formatTotal" placeholder="0.00" @keydown.enter="showSummary = true" />
    </div>

    <button @click="calculateSplit" class="btn btn-primary btn-block my-4" :disabled="persons.length === 0">Calculate Split</button>

    <div v-if="showSummary" class="mt-8 pt-4 border-t border-gray-300">
      <h2 class="text-2xl font-bold text-center">Payment Summary</h2>
      <h4 class="text-center mb-4">"Total Payment" includes each person's share of tip and tax.</h4>
      <div class="overflow-x-auto">
        <table class="table w-4/5 mx-auto">
          <thead>
          <tr>
            <th>Person</th>
            <th>Total Payment</th>
          </tr>
          </thead>
          <tbody>
          <tr v-for="(amount, person) in finalTotals" :key="person">
            <td class="font-medium">{{ person }}</td>
            <td>${{ amount }}</td>
          </tr>
          </tbody>
        </table>
      </div>
    </div>
<!--  Start details section. We're adding this later  -->
<!--    <div v-if="showSummary" class="mt-8 pt-4 border-t border-gray-300">-->
<!--      <h2 class="text-2xl font-bold text-center">Payment Summary</h2>-->
<!--      <h4 class="text-center mb-4">"Total Payment" includes each person's share of tip and tax.</h4>-->

<!--      <div class="space-y-6">-->
<!--        <div v-for="person in persons" :key="person" class="card card-bordered bg-base-200">-->
<!--          <div class="card-body">-->
<!--            <h3 class="text-xl font-semibold mb-2">{{ person }}</h3>-->
<!--            <ul class="list-disc list-inside mb-2">-->
<!--              <li v-for="entry in personBreakdown[person]" :key="entry.item">-->
<!--                {{ entry.item }} — ${{ entry.share }}-->
<!--              </li>-->
<!--            </ul>-->
<!--            <p class="font-bold text-right">Total: ${{ finalTotals[person] }}</p>-->
<!--          </div>-->
<!--        </div>-->
<!--      </div>-->
<!--    </div>-->
<!--  end details section  -->
  </div>
</template>

***

<script setup>
import { ref, computed, onMounted, watch, nextTick } from 'vue'
import './app.css';
import { gsap } from "gsap"

// --- STATE ---
const inDevMode = ref(import.meta.env.DEV);

const persons = ref([]);
const newPersonName = ref('');

const progressBar = ref(null)
let timeline = null

const billItems = ref([]);
const newMenuItemName = ref('');
const showManualItemInput = ref(false);

const totalAmount = ref('');
const showSummary = ref(false);

const showTipPopup = ref(false);
const dontShowAgain = ref(false);

const uploadError = ref('');
const isUploading = ref(false);
let nextItemId = 0;

// --- COMPUTED PROPERTIES (Derived State) ---
const subtotal = computed(() => {
  return billItems.value.reduce((total, item) => total + (parseFloat(item.price) || 0), 0);
});

/**
 * Calculates the final payment for each person, including their share of tax and tip.
 */
const finalTotals = computed(() => {
  const sub = subtotal.value;
  const total = parseFloat(totalAmount.value) || sub; // Default to subtotal if total isn't entered
  if (sub === 0 || persons.value.length === 0) return {};

  const multiplier = total / sub; // The ratio of total to subtotal (for tax/tip)

  // Initialize a total for each person
  const personTotals = persons.value.reduce((acc, p) => ({ ...acc, [p]: 0 }), {});

  // Distribute the cost of each item
  billItems.value.forEach(item => {
    const price = parseFloat(item.price) || 0;
    if (price === 0) return;

    // If no one is checked for an item, split it among everyone. Otherwise, use checked people.
    const peopleSharing = item.splitWith.length > 0 ? item.splitWith : persons.value;
    if (peopleSharing.length === 0) return;

    const share = price / peopleSharing.length;
    peopleSharing.forEach(person => {
      if (personTotals[person] !== undefined) personTotals[person] += share;
    });
  });

  // Apply the tax/tip multiplier to each person's total and format it
  const final = {};
  for (const person in personTotals) {
    final[person] = (personTotals[person] * multiplier).toFixed(2);
  }
  return final;
});

/**
 * Builds an itemized breakdown for each person.
 */
const personBreakdown = computed(() => {
  const breakdown = {};
  const sub = subtotal.value;
  const total = parseFloat(totalAmount.value) || sub;
  if (sub === 0 || persons.value.length === 0) return breakdown;

  const multiplier = total / sub; // tax/tip multiplier

  persons.value.forEach(p => { breakdown[p] = []; });

  billItems.value.forEach(item => {
    const price = parseFloat(item.price) || 0;
    if (!price) return;

    const sharers = item.splitWith.length > 0 ? item.splitWith : persons.value;
    const share = (price / sharers.length) * multiplier;

    sharers.forEach(p => {
      breakdown[p].push({
        item: item.name || "Unnamed Item",
        share: share.toFixed(2)
      });
    });
  });

  return breakdown;
});

// --- METHODS ---
function startProgressBarAnimation() {
  // Kill old animation if running
  if (timeline) {
    timeline.kill()
    progressBar.value.value = 0
  }

  timeline = gsap.timeline()

  // Phase 1: 0 → 80% in 9s
  timeline.to(progressBar.value, {
    value: 95,
    duration: 20,
    ease: "power2.out"
  })
}

function finishProgressBarAnimation() {
  // if no timeline, return. else, kill it
  if (!timeline) return
  timeline.kill()

  // Jump to end of current tween and animate quickly to 100%
  gsap.to(progressBar.value, {
    value: 100,
    duration: 0.2,
    ease: "linear",
  })

  focusPeopleInput();
}

function focusPeopleInput() {
  // scroll Add People section to top of page
  document.querySelector('#add-people-section').scrollIntoView({
    behavior: 'smooth',
    block: 'start'   // align the element to the top of the viewport
  });

  // focus input field
  document.getElementById('person-input').focus()
}

/**
 * Adds one or multiple people from a single space-separated string.
 */
function handleAddPerson() {
  const names = newPersonName.value.trim().split(/\s+/).filter(Boolean);
  names.forEach(name => {
    if (!persons.value.includes(name)) {
      persons.value.push(name);
    }
  });
  newPersonName.value = '';
}

async function calculateSplit() {
  showSummary.value = true;
  await nextTick();
  window.scrollTo({
    top: document.documentElement.scrollHeight,
    behavior: 'smooth'
  });
}

function togglePersonForItem(item, person) {
  const index = item.splitWith.indexOf(person);
  if (index > -1) {
    // Person is in the array, so remove them
    item.splitWith.splice(index, 1);
  } else {
    // Person is not in the array, so add them
    item.splitWith.push(person);
  }

  handleShouldShowTip(item)
}

function handleShouldShowTip(item) {
  console.log('hi')
  if (item.splitWith.length === persons.value.length) {
    console.log('should show')
    showPopup();
  }
}

/**
 * Adds a new, empty row to the bill items table.
 */
function addItemRow(name = '', price = '') {
  billItems.value.push({
    id: nextItemId++,
    name,
    price: price.toString(),
    splitWith: [] // This array will be populated by the checkboxes via v-model
  });
}

/**
 * Adds multiple items from a comma or period separated string.
 */
function addMenuItem() {
  const items = newMenuItemName.value.split(/[,.]/).map(item => item.trim()).filter(Boolean);
  items.forEach(item => addItemRow(item));
  newMenuItemName.value = '';
  // If the table was empty, add an extra blank row for convenience
  if (billItems.value.length === items.length && items.length > 0) {
    addItemRow();
  }
}

/**
 * Removes an item from the bill by its unique ID.
 */
function removeItem(itemId) {
  billItems.value = billItems.value.filter(item => item.id !== itemId);
}

/**
 * Formats a number input to two decimal places automatically.
 */
function formatPrice(item, event) {
  const value = event.target.value.replace(/[^0-9]/g, '');
  item.price = value ? (parseInt(value, 10) / 100).toFixed(2) : '';
}

/**
 * Formatter for the main Total field.
 */
function formatTotal(event) {
  const value = event.target.value.replace(/[^0-9]/g, '');
  totalAmount.value = value ? (parseInt(value, 10) / 100).toFixed(2) : '';
}

/**
 * Closes the informational popup and saves the user's preference if checked.
 */
function closePopup() {
  if (dontShowAgain.value) {
    localStorage.setItem('dontShowPopup', 'true');
  }
  showTipPopup.value = false;
}

/**
 * Handles the file upload, compression, and API call for receipt parsing.
 */
async function handleReceiptUpload(event) {
  // get file
  const file = event.target.files[0];
  if (!file) return;

  startProgressBarAnimation();

  uploadError.value = '';
  isUploading.value = true;

  try {
    if (file.size > 5 * 1024 * 1024) {
      throw new Error('File is too large. Max 5MB.');
    }

    const compressedFile = await compressImage(file);
    const formData = new FormData();
    formData.append('image', compressedFile);

    const response = await fetch('https://receipt-parser.kenqiwu-1b0.workers.dev/parse_receipt/', {
      method: 'POST',
      body: formData
    });

    // if response not ok, throw errow
    if (!response.ok) throw new Error('Failed to parse receipt. Please try again.');

    // wait on data and then finish progress bar
    const data = await response.json();
    finishProgressBarAnimation();

    // Clear existing items and populate with parsed data
    billItems.value = [];
    data.line_items.forEach(item => addItemRow(item.name, item.price.toFixed(2)));
    if (data.total_amount) totalAmount.value = data.total_amount.toFixed(2);
  } catch (err) {
    uploadError.value = err.message;
  } finally {
    isUploading.value = false;
    event.target.value = ''; // Reset file input
  }
}

/**
 * Compresses an image file using a canvas before uploading.
 */
function compressImage(file, quality = 0.8) {
  return new Promise((resolve, reject) => {
    const reader = new FileReader();
    reader.readAsDataURL(file);
    reader.onload = event => {
      const img = new Image();
      img.src = event.target.result;
      img.onload = () => {
        const canvas = document.createElement('canvas');
        canvas.width = img.width;
        canvas.height = img.height;
        const ctx = canvas.getContext('2d');
        ctx.drawImage(img, 0, 0);
        canvas.toBlob(blob => resolve(new File([blob], file.name, { type: 'image/jpeg' })), 'image/jpeg', quality);
      };
      img.onerror = reject;
    };
    reader.onerror = reject;
  });
}

function showPopup() {
  showTipPopup.value = true;
}

// --- WATCHERS (Reacting to State Changes) ---

/**
 * When the list of persons changes (e.g., someone is removed), this ensures they
 * are also removed from any items they were splitting.
 */
watch(persons, (newPersonsList) => {
  const newPersonsSet = new Set(newPersonsList);
  billItems.value.forEach(item => {
    item.splitWith = item.splitWith.filter(p => newPersonsSet.has(p));
  });
}, { deep: true });

// --- LIFECYCLE HOOKS ---

/**
 * Runs once after the component is mounted to the DOM.
 */
onMounted(() => {
  // Check if the user has previously opted out of the tip popup.
  if (!localStorage.getItem('dontShowPopup')) {
    showTipPopup.value = true;
  }
  // Start with one empty row for the user.
  addItemRow();
});
</script>

***

<style>
body {
  font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Helvetica, Arial, sans-serif;
  padding-top: var(--safe-area-inset-top);
  padding-right: var(--safe-area-inset-right);
  padding-bottom: var(--safe-area-inset-bottom);
  padding-left: var(--safe-area-inset-left);
}
.bill-splitter-app {
  max-width: 800px;
  margin: auto;
  padding: 6px;
  border-radius: 4px;
}

:root {
  --safe-area-inset-top: env(safe-area-inset-top);
  --safe-area-inset-right: env(safe-area-inset-right);
  --safe-area-inset-bottom: env(safe-area-inset-bottom);
  --safe-area-inset-left: env(safe-area-inset-left);
}
</style>
