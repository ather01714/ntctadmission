 
// --- STATE MANAGEMENT ---
let currentFormData = {};
let currentPhotoBase64 = "";

// Populate Year Dropdown (1980 - 2030)
window.onload = function() {
    const yearSelect = document.getElementById("passingYear");
    for (let i = 1980; i <= 2030; i++) {
        let option = document.createElement("option");
        option.value = i;
        option.text = i;
        yearSelect.add(option);
    }
   
    // Auto-Uppercase Listener
    document.querySelectorAll("input, select").forEach(el => {
        el.addEventListener("input", function() {
            if(this.type === 'text') {
                this.value = this.value.toUpperCase();
            }
        });
    });
};

// --- NAVIGATION ROUTER ---
function router(viewId) {
    // Hide all views
    document.querySelectorAll('.view').forEach(view => {
        view.classList.remove('active');
        view.classList.add('hidden');
    });
    // Show target view
    const target = document.getElementById(viewId);
    target.classList.remove('hidden');
    target.classList.add('active');
}

// --- STEP 1 LOGIC ---
function validateStep1() {
    // Collect Data
    const inputs = document.querySelectorAll('#step-1 input, #step-1 select');
    let isValid = true;
    currentFormData = {};

    inputs.forEach(input => {
        if(input.required && !input.value) {
            input.style.border = "1px solid red";
            isValid = false;
        } else {
            input.style.border = "1px solid #ddd";
            currentFormData[input.id] = input.value;
        }
    });

    // Specific Validations
    const idNum = document.getElementById('idNumber').value;
    const mobile = document.getElementById('mobile').value;
   
    // NID Validation (17 digits)
    if (!/^\d{10,17}$/.test(idNum)) {
        document.getElementById('idError').innerText = "Must be between 10 to 17 digits";
        isValid = false;
    } else {
        document.getElementById('idError').innerText = "";
    }

    // Mobile Validation (11 digits)
    if (!/^\d{11}$/.test(mobile)) {
        document.getElementById('mobileError').innerText = "Must be exactly 11 digits";
        isValid = false;
    } else {
        document.getElementById('mobileError').innerText = "";
    }

    if (isValid) {
        populatePreview();
        router('step-2');
    } else {
        alert("Please fix the errors marked in red.");
    }
}

// --- STEP 2 LOGIC ---
function previewImage(input) {
    if (input.files && input.files[0]) {
        const reader = new FileReader();
        reader.onload = function(e) {
            document.getElementById('imgPreview').src = e.target.result;
            document.getElementById('imgPreview').style.display = 'block';
            currentPhotoBase64 = e.target.result;
        }
        reader.readAsDataURL(input.files[0]);
    }
}

function populatePreview() {
    const table = document.getElementById('previewTable');
    let html = '';
    for (const [key, value] of Object.entries(currentFormData)) {
        // Beautify labels based on ID
        let label = key.replace(/([A-Z])/g, ' $1').trim();
        html += `<div class="preview-row">
                    <div class="preview-label">${label.toUpperCase()}</div>
                    <div class="preview-val">${value}</div>
                 </div>`;
    }
    table.innerHTML = html;
}

// --- SUBMISSION ---
function submitApplication() {
    if (!currentPhotoBase64) {
        alert("Please upload a photo.");
        return;
    }

    // Generate Application ID: YY-RRR (Year - Random 3)
    const year = new Date().getFullYear().toString().substr(-2);
    const random = Math.floor(Math.random() * 900 + 100);
    const appId = `${year}${random}`;
   
    currentFormData['appId'] = appId;
    currentFormData['photo'] = currentPhotoBase64;
    currentFormData['submitDate'] = new Date().toLocaleDateString();

    // Save to LocalStorage (Simulating Database)
    let applications = JSON.parse(localStorage.getItem('admission_apps')) || [];
    applications.push(currentFormData);
    localStorage.setItem('admission_apps', JSON.stringify(applications));

    router('success-view');
}

// --- CLIENT PDF GENERATOR ---
function downloadClientPDF() {
    const data = currentFormData;
   
    // Build PDF HTML Content
    const infoHtml = `
        <div class="pdf-line"><div class="pdf-label">Application ID:</div> ${data.appId}</div>
        <div class="pdf-line"><div class="pdf-label">Course:</div> ${data.courseTitle}</div>
        <div class="pdf-line"><div class="pdf-label">Time:</div> ${data.courseTime}</div>
        <div class="pdf-line"><div class="pdf-label">Season:</div> ${data.season}</div>
        <div class="pdf-line"><div class="pdf-label">Student Name:</div> ${data.nameEn}</div>
        <div class="pdf-line"><div class="pdf-label">Father Name:</div> ${data.fatherName}</div>
        <div class="pdf-line"><div class="pdf-label">Address:</div> ${data.presentAddr}</div>
        <div class="pdf-line"><div class="pdf-label">DOB:</div> ${data.dob}</div>
        <div class="pdf-line"><div class="pdf-label">Roll No:</div> ${data.rollNo}</div>
        <div class="pdf-line"><div class="pdf-label">Reg No:</div> ${data.regNo}</div>
        <div class="pdf-line"><div class="pdf-label">Board:</div> ${data.board}</div>
        <div class="pdf-line"><div class="pdf-label">ID Type:</div> ${data.idType}</div>
        <div class="pdf-line"><div class="pdf-label">ID No:</div> ${data.idNumber}</div>
        <div class="pdf-line"><div class="pdf-label">Mobile:</div> ${data.mobile}</div>
        <div class="pdf-line"><div class="pdf-label">Date:</div> ${data.submitDate}</div>
    `;

    document.querySelector('#client-pdf-template .pdf-info-left').innerHTML = infoHtml;
    document.getElementById('pdf-client-img').src = data.photo;

    const element = document.getElementById('client-pdf-template');
    element.style.display = 'block'; // Make visible for render
   
    const opt = {
        margin: 0.5,
        filename: `Application_${data.appId}.pdf`,
        image: { type: 'jpeg', quality: 0.98 },
        html2canvas: { scale: 2 },
        jsPDF: { unit: 'in', format: 'letter', orientation: 'portrait' }
    };

    html2pdf().from(element).set(opt).save().then(() => {
        element.style.display = 'none'; // Hide again
    });
}

// --- ADMIN PANEL ---
function checkAdminLogin() {
    const pass = document.getElementById('adminPass').value;
    if (pass === "arbin@50744") {
        document.getElementById('adminPass').value = ""; // clear
        loadAdminDashboard();
        router('admin-dashboard');
    } else {
        alert("Incorrect Password!");
    }
}

function loadAdminDashboard() {
    const apps = JSON.parse(localStorage.getItem('admission_apps')) || [];
    const tbody = document.getElementById('adminTableBody');
    tbody.innerHTML = "";

    apps.forEach((app, index) => {
        let row = `<tr>
            <td>${app.appId}</td>
            <td>${app.nameEn}</td>
            <td>${app.mobile}</td>
            <td>${app.courseTitle}</td>
            <td>${app.photo}</td>
            <td>
                <button class="btn-primary" style="padding:5px; font-size:12px;" onclick="downloadAdminPDF(${index})">
                    <i class="fas fa-file-pdf"></i> Download
                </button>
            </td>
        </tr>`;
        tbody.innerHTML += row;
    });
}

function downloadAdminPDF(index) {
    const apps = JSON.parse(localStorage.getItem('admission_apps'));
    const data = apps[index];

    const infoHtml = `
        <div class="pdf-line"><div class="pdf-label">Application ID:</div> ${data.appId}</div>
        <div class="pdf-line"><div class="pdf-label">Submission Date:</div> ${data.submitDate}</div>
        <hr style="margin: 10px 0;">
        <div class="pdf-line"><div class="pdf-label">Course Name:</div> ${data.courseTitle}</div>
        <div class="pdf-line"><div class="pdf-label">Course Time:</div> ${data.courseTime}</div>
        <div class="pdf-line"><div class="pdf-label">Season:</div> ${data.season}</div>
        <div class="pdf-line"><div class="pdf-label">Student Name (BN):</div> ${data.nameBn}</div>
        <div class="pdf-line"><div class="pdf-label">Student Name (EN):</div> ${data.nameEn}</div>
        <div class="pdf-line"><div class="pdf-label">Father's Name:</div> ${data.fatherName}</div>
        <div class="pdf-line"><div class="pdf-label">Mother's Name:</div> ${data.motherName}</div>
        <div class="pdf-line"><div class="pdf-label">Present Address:</div> ${data.presentAddr}</div>
        <div class="pdf-line"><div class="pdf-label">Permanent Address:</div> ${data.permAddr}</div>
        <div class="pdf-line"><div class="pdf-label">Date of Birth:</div> ${data.dob}</div>
        <div class="pdf-line"><div class="pdf-label">Roll Number:</div> ${data.rollNo}</div>
        <div class="pdf-line"><div class="pdf-label">Registration No:</div> ${data.regNo}</div>
        <div class="pdf-line"><div class="pdf-label">Passing Year:</div> ${data.passingYear}</div>
        <div class="pdf-line"><div class="pdf-label">Board:</div> ${data.board}</div>
        <div class="pdf-line"><div class="pdf-label">Religion:</div> ${data.religion}</div>
        <div class="pdf-line"><div class="pdf-label">Gender:</div> ${data.gender}</div>
        <div class="pdf-line"><div class="pdf-label">Mobile Number:</div> ${data.mobile}</div>
    `;

    document.querySelector('#admin-pdf-template .pdf-info-full').innerHTML = infoHtml;
    document.getElementById('pdf-admin-img').src = data.photo;

    const element = document.getElementById('admin-pdf-template');
    element.style.display = 'block';
   
    const opt = {
        margin: 0.5,
        filename: `AdminCopy_${data.appId}.pdf`,
        image: { type: 'jpeg', quality: 0.98 },
        html2canvas: { scale: 2 },
        jsPDF: { unit: 'in', format: 'a4', orientation: 'portrait' }
    };

    html2pdf().from(element).set(opt).save().then(() => {
        element.style.display = 'none';
    });
} 
