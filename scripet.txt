<script type="module">
    import { initializeApp } from "https://www.gstatic.com/firebasejs/10.7.1/firebase-app.js";
    import { 
        getAuth, 
        onAuthStateChanged, 
        signInWithPopup, 
        signInWithRedirect,
        getRedirectResult,
        GoogleAuthProvider, 
        signOut 
    } from "https://www.gstatic.com/firebasejs/10.7.1/firebase-auth.js";
    import { 
        getFirestore, 
        doc, 
        setDoc, 
        getDoc, 
        updateDoc, 
        collection, 
        onSnapshot,    
        getDocs, 
        addDoc, 
        deleteField, 
        deleteDoc,
        query, 
        where, 
        collectionGroup, 
        arrayUnion,      
        arrayRemove      
    } from "https://www.gstatic.com/firebasejs/10.7.1/firebase-firestore.js";

    // Firebase Configuration
    const firebaseConfig = { 
        apiKey: "AIzaSyAxDQCEsgzJU9L2XJPj5jeLPN6CV2lR3EY", 
        authDomain: "smartproduction-erp.firebaseapp.com", 
        projectId: "smartproduction-erp", 
        appId: "1:794138225480:web:1be841f305c5fe6e12617d" 
    };
    
    const app = initializeApp(firebaseConfig);
    const db = getFirestore(app);
    const auth = getAuth(app);
    const prov = new GoogleAuthProvider();

    // Global Variables
    let user = null, myChart = null, avgPieChart = null, persistentSectionID = "";
    let selectedMonth = (new Date().getMonth() + 1);
    const monthNames = ["Jan", "Feb", "Mar", "Apr", "May", "Jun", "Jul", "Aug", "Sep", "Oct", "Nov", "Dec"];
    const palette = ["#00bcd4", "#ff9800", "#e91e63", "#4caf50", "#9c27b0", "#3f51b5", "#f44336", "#795548", "#607d8b", "#ff5722", "#009688", "#ffc107"];

    // গ্লোবাল ভেরিয়েবল
    window.currentSecOwner = null; 
    window.currentSecID = "";
    window.isMasterMode = false;
    window.isRendering = false;
    window.allSections = [];

    // UI Helpers
    const showSpinner = (s) => {
        const spinner = document.getElementById('loadingSpinner');
        if(spinner) spinner.style.display = s ? 'flex' : 'none';
    }

    // Auth State Observer
    onAuthStateChanged(auth, u => {
        user = u;
        const authContainer = document.getElementById('auth-container');
        const landingPage = document.getElementById('landingPage');
        const appDashboard = document.getElementById('appDashboard');
        
        if(u) {
            if(landingPage) landingPage.style.display = 'none';
            if(appDashboard) {
                appDashboard.style.display = 'flex';
                appDashboard.classList.remove('hidden');
            }

            if(authContainer) {
                authContainer.innerHTML = `
                    <div class="user-profile-chip" onclick="logoutUser()" title="Click to Logout Immediately">
                        <img src="${u.photoURL}" alt="User">
                        <div class="user-info-text no-print">
                            <span class="u-name-text">${u.displayName}</span>
                            <span class="u-email-text">${u.email}</span>
                        </div>
                    </div>
                `;
            }
            setupFilters(); 
            loadDashboard();
        } else {
            if(landingPage) landingPage.style.display = 'block';
            if(appDashboard) {
                appDashboard.style.display = 'none';
                appDashboard.classList.add('hidden');
            }

            if(authContainer) {
                authContainer.innerHTML = `
                    <button onclick="loginUser()" style="background:#4285F4; color:white; border:none; padding:10px 22px; border-radius:30px; cursor:pointer; font-weight:600; display:flex; align-items:center; gap:10px; box-shadow: 0 4px 10px rgba(66,133,244,0.25);">
                        <svg width="18" height="18" viewBox="0 0 24 24" fill="white"><path d="M12.48 10.92v3.28h7.84c-.24 1.84-.909 3.292-2.09 4.413-1.177 1.108-2.81 2.03-5.75 2.03-4.574 0-8.38-3.714-8.38-8.288 0-4.574 3.806-8.288 8.38-8.288 2.504 0 4.41.986 5.76 2.276l2.314-2.314C18.497 2.012 15.75 1 12.48 1 6.645 1 2 5.645 2 11.48s4.645 10.48 10.48 10.48c3.15 0 5.534-1.036 7.425-3.008 1.95-1.95 2.57-4.7 2.57-6.95 0-.46-.04-.91-.11-1.35h-9.91z"/></svg>
                        Sign In with Google
                    </button>
                `;
            }
        }
    });

    // Authentication Functions (Updated to handle Popup Blocker)
    window.loginUser = () => {
        signInWithPopup(auth, prov).catch(err => {
            if (err.code === 'auth/popup-blocked' || err.code === 'auth/cancelled-popup-request') {
                console.warn("Popup blocked, trying redirect...");
                signInWithRedirect(auth, prov);
            } else {
                console.error("Login Error:", err);
            }
        });
    };

    // Handle Redirect Result
    getRedirectResult(auth).catch(err => console.error("Redirect Error:", err));

    window.logoutUser = () => signOut(auth).catch(err => console.error("Logout Error:", err));

    // Navigation & Modal Functions
    window.toggleSidebar = (e) => { 
        e.stopPropagation(); 
        const sidebar = document.getElementById('sidebar');
        if(sidebar) sidebar.classList.toggle('collapsed'); 
    };

  window.handleFilterUpdate = () => { 
    loadDashboard(); 
    
    // আপনার আগের লজিক
    if (document.getElementById('reportView') && !document.getElementById('reportView').classList.contains('hidden')) updatePresentation(); 
    if (document.getElementById('analyticsView') && !document.getElementById('analyticsView').classList.contains('hidden')) loadAnalytics();

    // এটি আপডেট করুন
    const selections = {
        Sizing: false,
        Month: selectedMonth, // বর্তমানে কোন মাস সিলেক্ট করা আছে
        User: user ? user.displayName : 'Guest' // কে আপডেট করছে
    };

    console.log("Selections updated:", selections);
};

    // --- ট্যাব এবং মডাল ফাংশনসমূহ ---
    window.switchTab = (t, e) => { 
        document.querySelectorAll('.nav-item').forEach(n => n.classList.remove('active')); 
        if(e) e.currentTarget.classList.add('active'); 
        
        const dashboardView = document.getElementById('dashboardView');
        const reportView = document.getElementById('reportView');
        const analyticsView = document.getElementById('analyticsView');
        const fSecFilter = document.getElementById('fSecFilter');
        const topAction = document.getElementById('dashboardTopAction');

        if(dashboardView) dashboardView.style.display = (t === 'dashboard' ? 'flex' : 'none');
        if(reportView) reportView.classList.toggle('hidden', t !== 'report');
        if(analyticsView) analyticsView.classList.toggle('hidden', t !== 'analytics');
        if(fSecFilter) fSecFilter.classList.toggle('hidden', t !== 'report'); 

        if (topAction) topAction.style.display = (t === 'dashboard' ? 'block' : 'none');

        const viewTitle = document.getElementById('viewTitle');
        if(viewTitle) {
            viewTitle.innerText = 
                t === 'dashboard' ? 'Dashboard' : 
                t === 'report' ? 'KPI Report' : 
                t === 'analytics' ? 'Analytics' : '';
        }

        if(t === 'report' && typeof updatePresentation === 'function') updatePresentation(); 
        if(t === 'analytics' && typeof loadAnalytics === 'function') loadAnalytics();
    };

   // --- Function to open popup on dashboard card click ---
    window.openEntryModal = (id, name, owner) => { 
        window.currentSecID = id;
        window.currentSecOwner = owner; 
        const modal = document.getElementById('entryModal');
        if(modal) {
            modal.style.display = 'block';
            if(document.getElementById('entryModalTitle')) {
                document.getElementById('entryModalTitle').innerText = `Data Entry: ${name}`;
            }
            if(document.getElementById('prodDate')) {
                document.getElementById('prodDate').valueAsDate = new Date();
            }
        }
        if (typeof loadSectionData === 'function') loadSectionData(id, owner);
    };

    window.openModal = (id) => { 
        const modal = document.getElementById(id);
        if(modal) {
            modal.style.display = 'block';
            if(id === 'entryModal' && document.getElementById('prodDate')) {
                document.getElementById('prodDate').valueAsDate = new Date();
            }
        }
    };

    window.closeModal = (id) => { 
        const modal = document.getElementById(id);
        if(modal) modal.style.display = 'none'; 
        window.isMasterMode = false; 
        const masterActions = document.getElementById('masterActions');
        if(masterActions) masterActions.classList.add('hidden'); 
    };

    window.closeAllMenus = () => { 
        document.querySelectorAll('.dot-menu').forEach(m => m.style.display = 'none'); 
    };

    window.toggleDotMenu = (e, id) => { 
        e.stopPropagation(); 
        closeAllMenus(); 
        const menu = document.getElementById(`menu-${id}`);
        if(menu) menu.style.display = 'block'; 
    };

    window.saveData = async () => {
    // 1. Find the current section and its data
    const section = allSections.find(s => s.id === currentSecID);
    if (!section) {
        console.error("Section not found!");
        return;
    }

    // 2. Permission Check
    const isOwner = section.owner === user.email;
    const userKey = user.email.replace(/\./g, '_dot_'); 
    
    // Role check logic
    let userRole = 'viewer'; 
    if (isOwner) {
        userRole = 'admin';
    } else if (section.roles && section.roles[userKey]) {
        // Converting the role from database to lowercase
        userRole = section.roles[userKey].toLowerCase();
    }

    // 3. Strict Check: If user is not 'admin', 'editor', or 'owner', save will not proceed
    const hasWriteAccess = isOwner || userRole === 'admin' || userRole === 'editor';

    if (!hasWriteAccess) {
        Swal.fire({
            icon: 'error',
            title: 'No Permission!',
            text: 'You are a viewer. You do not have permission to save data.',
            confirmButtonColor: '#d33'
        });
        return; 
    }

    // 4. Data will be saved only if access is granted
    showSpinner(true);
    try {
        const date = document.getElementById('prodDate').value;
        if (!date) {
            Swal.fire('Error', 'Please select a date', 'error');
            showSpinner(false);
            return;
        }

        const tables = typeof getCurrentTables === 'function' ? getCurrentTables() : [];
        const payload = {
            Target: Number(document.getElementById('tVal').value) || 0,
            Achieved: Number(document.getElementById('aVal').value) || 0,
            Note: document.getElementById('prodNote')?.value || "",
            tables: tables,
            updatedAt: new Date().toISOString()
        };

        // Save data to the owner's path
        const dailyRef = doc(db, "user_data", section.owner, "sections", currentSecID, "daily_data", date);
        await setDoc(dailyRef, payload, { merge: true });

        // Update master configuration (Only for owner or admin)
        if (isMasterMode && (isOwner || userRole === 'admin')) {
            const secRef = doc(db, "user_data", section.owner, "sections", currentSecID);
            await setDoc(secRef, { masterConfig: tables }, { merge: true });
        }

        closeModal('entryModal');
        await loadDashboard(); 
        Swal.fire('Success', 'Data saved successfully.', 'success');

    } catch (e) {
        console.error("Save error details:", e);
        // Display error if rejected by Firebase rules
        Swal.fire('Error', 'Firebase Permission Error: You do not have permission to save this data.', 'error');
    } finally {
        showSpinner(false);
    }
};

    // --- Filter Setup ---
    function setupFilters() {
        const yearSelect = document.getElementById('fYear');
        if(!yearSelect) return;
        yearSelect.innerHTML = "";
        const curYear = new Date().getFullYear();
        for(let i = 0; i < 3; i++) {
            let opt = document.createElement('option');
            opt.value = curYear - i; 
            opt.innerText = curYear - i;
            yearSelect.appendChild(opt);
        }

        const monthCont = document.getElementById('monthContainer');
        if(!monthCont) return;
        monthCont.innerHTML = "";
        monthNames.forEach((m, i) => {
            const btn = document.createElement('div');
            btn.innerText = m;
            btn.style.cssText = `padding:5px 10px; cursor:pointer; border-radius:15px; font-size:12px; ${i + 1 === selectedMonth ? 'background:var(--primary); color:white;' : ''}`;
            btn.onclick = () => {
                selectedMonth = i + 1;
                Array.from(monthCont.children).forEach(c => { c.style.background = "none"; c.style.color = "black"; });
                btn.style.background = "var(--primary)"; btn.style.color = "white";
                handleFilterUpdate();
            };
            monthCont.appendChild(btn);
        });
    }
window.loadDashboard = async () => {
    if (!user) return;
    showSpinner(true);

    try {
        const year = document.getElementById('fYear').value;
        const monthEnabled = document.getElementById('monthEnable').checked;
        const prefix = monthEnabled ? `${year}-${selectedMonth.toString().padStart(2, '0')}` : `${year}`;

        if (typeof onSnapshot === 'undefined') {
            throw new Error("onSnapshot is not defined. Please check your imports.");
        }

        // Real-time listener (For user's own sections)
        onSnapshot(collection(db, "user_data", user.email, "sections"), async snap => {
            if (isRendering) return; 
            isRendering = true;

            const container = document.getElementById('dashboardView');
            if (!container) {
                isRendering = false;
                showSpinner(false);
                return;
            }

            let tempSections = [];
            snap.forEach(d => {
                if (!d.data().deleted) {
                    tempSections.push({ id: d.id, ...d.data(), owner: user.email });
                }
            });

            // Search for shared sections (Other's data)
            try {
                const qShared = query(
                    collectionGroup(db, "sections"), 
                    where("sharedWith", "array-contains", user.email)
                );
                const sharedSnap = await getDocs(qShared);
                sharedSnap.forEach(d => {
                    const data = d.data();
                    const ownerEmail = d.ref.path.split('/')[1]; 
                    if (!data.deleted && ownerEmail !== user.email) {
                        tempSections.push({ id: d.id, ...data, owner: ownerEmail });
                    }
                });
            } catch (e) {
                console.warn("Shared access issue:", e);
            }

            // Remove duplicates and sorting
            const uniqueSections = [];
            const seenIds = new Set();
            tempSections.forEach(s => {
                if (!seenIds.has(s.id)) {
                    uniqueSections.push(s);
                    seenIds.add(s.id);
                }
            });
            
            // Update global variable (Ensures other functions get data)
            window.allSections = uniqueSections.sort((a, b) => {
                const nameA = (a.sectionName || a.name || "").toLowerCase();
                const nameB = (b.sectionName || b.name || "").toLowerCase();
                return nameA.localeCompare(nameB, undefined, {numeric: true, sensitivity: 'base'});
            });
            
            updateDropdown();

            container.innerHTML = ""; 
            for (const sec of allSections) {
                const displayName = sec.sectionName || sec.name || "Untitled";
                await renderDashCard(sec.id, displayName, prefix, sec.owner);
            }

            if (typeof window.updatePresentation === "function") {
                await window.updatePresentation();
            }

            showSpinner(false);
            isRendering = false;
        });
    } catch (criticalError) {
        console.error("Critical Load Error:", criticalError);
        showSpinner(false);
    }
};

async function renderDashCard(id, name, prefix, owner) {
    try {
        const dailySnap = await getDocs(collection(db, "user_data", owner, "sections", id, "daily_data"));
        let t = 0, a = 0;
        dailySnap.forEach(day => { 
            if(day.id.startsWith(prefix)) { 
                t += Number(day.data().Target)||0; 
                a += Number(day.data().Achieved)||0; 
            } 
        });
        
        const p = t > 0 ? Math.round((a/t)*100) : 0;
        const section = allSections.find(s => s.id === id);
        const cardIdx = allSections.findIndex(s => s.id === id);
        const sectionColor = palette[cardIdx % palette.length] || '#444';
        
        const isOwner = owner === user.email;
        const userKey = user.email.replace(/\./g, '_dot_');
        
        // Determine role
        let userRole = 'viewer'; 
        if (isOwner) {
            userRole = 'admin';
        } else if (section && section.roles && section.roles[userKey]) {
            userRole = section.roles[userKey].toLowerCase();
        }

        const lT = section?.labelT || "Target";
        const lA = section?.labelA || "Achieved";
        const lB = section?.labelB || "Balance";

        const div = document.createElement('div'); 
        div.className = "card";
        div.style.cssText = "animation: fadeIn 0.6s ease; cursor:pointer;";
        
        div.onclick = () => { 
            if (userRole === 'viewer') {
                return Swal.fire({
                    icon: 'error',
                    title: 'Access Denied',
                    text: 'You are a viewer; you do not have permission to enter data.',
                    confirmButtonColor: '#d33'
                });
            }
            window.isMasterMode = (userRole === 'admin' || userRole === 'editor');
            const actions = document.getElementById('masterActions');
            if(actions) actions.classList.add('hidden'); 

            window.currentSecID = id; 
            window.currentSecOwner = owner; 
            
            const modalTitle = document.getElementById('modalSecName');
            if(modalTitle) modalTitle.innerText = name; 

            openModal('entryModal'); 
            if(typeof fetchDataByDate === 'function') fetchDataByDate();
        };

        // --- Menu Logic: Owners and Admins get Share options ---
        let menuHtml = "";
        if (isOwner || userRole === 'admin') {
            menuHtml += `<p onclick="openShareModal(event, '${id}', '${name}')">Share</p>`;
        }
        
        if (isOwner) {
            menuHtml += `<p onclick="deleteSection(event, '${id}')" style="color:red;">Delete Section</p>`;
        } else {
            menuHtml += `<p onclick="leaveSection(event, '${id}', '${owner}')" style="color:red;">Remove Access</p>`;
        }

        // Updated font-size to 18px for all elements below
        div.innerHTML = `
            <div class="three-dot" onclick="toggleDotMenu(event, '${id}')">⋮</div>
            <div id="menu-${id}" class="dot-menu">${menuHtml}</div>
            <div class="card-info">
                <h2 style="font-size:18px; font-weight:normal; color:${sectionColor}; margin-bottom:10px;">${name}</h2>
                <div class="stat-row" style="font-size:18px; color:#444; margin-bottom:5px;">
                    <span>${lT}</span><span class="colon">:</span><span>${t.toLocaleString('en-IN')}</span>
                </div>
                <div class="stat-row" style="font-size:18px; color:#444; margin-bottom:5px;">
                    <span>${lA}</span><span class="colon">:</span><span>${a.toLocaleString('en-IN')}</span>
                </div>
                <div class="stat-row" style="font-size:18px; color:#444;">
                    <span>${lB}</span><span class="colon">:</span><span>${(t-a < 0 ? 0 : t-a).toLocaleString('en-IN')}</span>
                </div>
            </div>
            <div class="circle-box">
                <svg><circle class="c-bg" cx="55" cy="55" r="45"></circle><circle class="c-val" cx="55" cy="55" r="45" style="stroke:${sectionColor}; stroke-dasharray:283; stroke-dashoffset:283; transition: stroke-dashoffset 1.5s ease-in-out;"></circle></svg>
                <div class="perc" style="font-size:18px; font-weight:normal; color:#444;">${p}%</div>
            </div>`;

        document.getElementById('dashboardView').appendChild(div);
        setTimeout(() => { 
            const circle = div.querySelector('.c-val'); 
            if(circle) circle.style.strokeDashoffset = 283 - (283 * p) / 100; 
        }, 200);
    } catch (err) { console.error("Card render error:", err); }
}

function updateDropdown() {
    const sel = document.getElementById('fSecFilter');
    if (!sel) return;
    const currentVal = sel.value;
    sel.innerHTML = allSections.map(s => {
        const dName = s.sectionName || s.name || "Untitled";
        return `<option value="${s.id}" ${s.id === currentVal ? 'selected' : ''}>${dName}</option>`;
    }).join('');
}

window.deleteSection = async (e, id) => {
    if (e) e.stopPropagation();
    const result = await Swal.fire({
        title: 'Are you sure?',
        text: "This section will be permanently deleted!",
        icon: 'warning',
        showCancelButton: true,
        confirmButtonColor: '#d33',
        cancelButtonColor: '#3085d6',
        confirmButtonText: 'Yes, delete it!'
    });
    
    if (!result.isConfirmed) return;
    showSpinner(true);
    
    try {
        await deleteDoc(doc(db, "user_data", user.email, "sections", id));
        await Swal.fire('Deleted!', 'Section has been deleted.', 'success');
        await window.loadDashboard();
    } catch (err) {
        console.error("Delete Error:", err);
        Swal.fire('Error!', 'Something went wrong.', 'error');
    } finally {
        showSpinner(false);
    }
};

window.leaveSection = async (event, id, owner) => {
    event.stopPropagation(); // Stop card click event
    
    const result = await Swal.fire({
        title: 'Are you sure?',
        text: "Do you want to remove your access from this section?",
        icon: 'warning',
        showCancelButton: true,
        confirmButtonColor: '#d33',
        cancelButtonColor: '#3085d6',
        confirmButtonText: 'Yes, remove it'
    });

    if (result.isConfirmed) {
        showSpinner(true);
        try {
            const userKey = user.email.replace(/\./g, '_dot_');
            const secRef = doc(db, "user_data", owner, "sections", id);

            // Remove user from database
            await updateDoc(secRef, {
                sharedWith: arrayRemove(user.email),
                [`roles.${userKey}`]: deleteField() // Remove from roles as well
            });

            Swal.fire('Removed!', 'Your access has been successfully removed.', 'success');
            loadDashboard(); // Refresh Dashboard
        } catch (e) {
            console.error("Leave error:", e);
            Swal.fire('Error!', 'Could not remove access. You might not be the owner or rules do not permit this.', 'error');
        } finally {
            showSpinner(false);
        }
    }
};
window.updatePresentation = async () => {
    showSpinner(true);
    try {
        let sid = document.getElementById('fSecFilter').value || (typeof persistentSectionID !== 'undefined' ? persistentSectionID : null);
        if(!sid) { showSpinner(false); return; }
        
        const section = allSections.find(s => s.id === sid);
        if(!section) { showSpinner(false); return; }
        
        persistentSectionID = sid; 
        const year = document.getElementById('fYear').value; 
        const monthEnabled = document.getElementById('monthEnable').checked;
        const prefix = monthEnabled ? `${year}-${selectedMonth.toString().padStart(2,'0')}` : `${year}`;
        
        const lT = section.labelT || "Target";
        const lA = section.labelA || "Achieved";
        const lB = section.labelB || "Balance";
        const savedLabel = section.contributionLabel || "Contribution"; 
        
        const dailySnap = await getDocs(collection(db, "user_data", section.owner, "sections", sid, "daily_data"));
        let totalT = 0, totalA = 0, mData = new Array(12).fill(0), monthItemTotals = {}, cumulativeT = 0, cumulativeA = 0;
        
        dailySnap.forEach(ds => {
            if(ds.id.startsWith(year)) {
                const m = parseInt(ds.id.split('-')[1]); 
                const mAchieved = (ds.data().Achieved||0); 
                const mTarget = (ds.data().Target||0);
                mData[m-1] += mAchieved; 
                if(m <= selectedMonth) { cumulativeT += mTarget; cumulativeA += mAchieved; }
                if(ds.id.startsWith(prefix)) {
                    totalT += mTarget; totalA += mAchieved;
                    if(ds.data().tables) { 
                        Object.entries(ds.data().tables).forEach(([tn, td]) => { 
                            if(!monthItemTotals[tn]) monthItemTotals[tn] = {}; 
                            Object.entries(td.items || {}).forEach(([iN, iV]) => {
                                monthItemTotals[tn][iN] = (monthItemTotals[tn][iN]||0) + iV;
                            }); 
                        }); 
                    }
                }
            }
        });

        const pCur = totalT > 0 ? Math.round((totalA/totalT)*100) : 0;
        const titleLabel = monthEnabled ? monthNames[selectedMonth-1] : year;

        const overviewCard = document.getElementById('overviewCard');
        if (overviewCard) {
            overviewCard.innerHTML = `
                <div class="card-info" onclick="showOverviewDetails()" style="cursor:pointer; width:100%; animation: fadeIn 0.8s ease;">
                    <h2 style="font-size:18px; font-weight:normal; color:#444;">${titleLabel} Total</h2>
                    <div class="stat-row" style="font-size:18px; color:#444;"><span>${lT}</span><span class="colon">:</span><span>${totalT.toLocaleString('en-IN')}</span></div>
                    <div class="stat-row" style="font-size:18px; color:#444;"><span>${lA}</span><span class="colon">:</span><span>${totalA.toLocaleString('en-IN')}</span></div>
                    <div class="stat-row" style="font-size:18px; color:#444;"><span>${lB}</span><span class="colon">:</span><span>${(totalT-totalA < 0 ? 0 : totalT-totalA).toLocaleString('en-IN')}</span></div>
                </div>
                <div class="circle-box">
                    <svg><circle class="c-bg" cx="55" cy="55" r="45"></circle><circle class="c-val" cx="55" cy="55" r="45" style="stroke-dasharray:283; stroke-dashoffset:283; transition: stroke-dashoffset 1.5s ease;"></circle></svg>
                    <div class="perc" style="font-size:18px; font-weight:normal; color:#444;">${pCur}%</div>
                </div>`;
            setTimeout(() => { 
                const circle = overviewCard.querySelector('.c-val'); 
                if(circle) circle.style.strokeDashoffset = 283 - (283 * pCur) / 100; 
            }, 200);
        }

        const avgA = Math.round(cumulativeA / selectedMonth) || 0; 
        const avgT = Math.round(cumulativeT / selectedMonth) || 0;
        const averageCard = document.getElementById('averageCard');
        if (averageCard) {
            averageCard.innerHTML = `
                <div class="card-info" style="animation: fadeIn 1s ease;">
                    <h2 style="font-size:18px; font-weight:normal; color:#444;">Monthly Avg</h2>
                    <div class="stat-row" style="font-size:18px; color:#444;"><span>${lT}</span><span class="colon">:</span><span>${totalT.toLocaleString('en-IN')}</span></div>
                    <div class="stat-row" style="font-size:18px; color:#444;"><span>${lA}</span><span class="colon">:</span><span>${avgA.toLocaleString('en-IN')}</span></div>
                    <div class="stat-row" style="font-size:18px; color:#444;"><span>${lB}</span><span class="colon">:</span><span>${((avgT-avgA) < 0 ? 0 : (avgT-avgA)).toLocaleString('en-IN')}</span></div>
                </div>
                <div class="pie-container"><canvas id="avgPieChartCanvas"></canvas></div>`;
            renderAvgPie(avgA, (avgT-avgA) < 0 ? 0 : (avgT-avgA));
        }

        renderChart(mData); 
        renderGroups(monthItemTotals, totalA, savedLabel); 

        showSpinner(false);
    } catch (e) {
        console.error("Error in updatePresentation:", e);
        showSpinner(false);
    }
};

// ১. গ্র্যান্ড টোটাল এবং পার্সেন্টেজ ক্যালকুলেশন ফাংশন
window.updateGrandTotal = function(shouldSave = true) {
    let newTotal = 0;
    const rows = document.querySelectorAll('.summary-row');
    const totalDisplay = document.getElementById('dynamic-grand-total');
    const selections = {};

    // টোটাল যোগ করা
    rows.forEach(row => {
        const checkbox = row.querySelector('.item-checkbox');
        const isChecked = checkbox && checkbox.checked;
        if (isChecked) {
            newTotal += parseFloat(row.getAttribute('data-value')) || 0;
        }
        selections[row.getAttribute('data-name')] = isChecked;
    });

    if (totalDisplay) totalDisplay.innerText = newTotal.toLocaleString('en-IN');

    // পার্সেন্টেজ (%) আপডেট করা
    rows.forEach((row) => {
        const val = parseFloat(row.getAttribute('data-value')) || 0;
        const checkbox = row.querySelector('.item-checkbox');
        const isChecked = checkbox && checkbox.checked;
        const bar = row.querySelector('.progress-bar-fill');
        const label = row.querySelector('.percentage-label');

        if (isChecked && newTotal > 0) {
            const p = Math.round((val / newTotal) * 100);
            if (label) label.innerText = p + '%';
            if (bar) bar.style.width = p + '%';
            row.style.opacity = "1";
        } else {
            if (label) label.innerText = '0%';
            if (bar) bar.style.width = '0%';
            row.style.opacity = "0.4";
        }
    });
    
    // ফায়ারবেস সেভ লজিক (প্রয়োজন হলে রাখবেন)
    console.log("Selections updated:", selections);
};

// ২. টাইটেল সেভ করার ফাংশন (যাতে saveContributionTitle এরর না আসে)
window.saveContributionTitle = async function(newTitle) {
    if(!persistentSectionID || typeof user === 'undefined' || !user) return;
    try {
        const { doc, updateDoc } = await import("https://www.gstatic.com/firebasejs/10.7.1/firebase-firestore.js");
        const secRef = doc(db, "user_data", user.email, "sections", persistentSectionID);
        await updateDoc(secRef, { contributionLabel: newTitle });
    } catch (e) {
        console.error("Title Save Error:", e);
    }
};

function renderGroups(items, monthlyGrandTotal, currentLabel) {
    const container = document.getElementById('presGroupContainer');
    if (!container) return;
    container.innerHTML = "";
    
    const section = allSections.find(s => s.id === persistentSectionID);
    const savedSelections = (section && section.contributionSelections) ? section.contributionSelections : {};
    
    let idx = 0; 
    let summaries = [];

    // প্রতিটি গ্রুপের ডাটা প্রসেস করা
    Object.entries(items).forEach(([tN, vals]) => {
        const sub = Object.values(vals).reduce((s, v) => s + v, 0);
        const color = palette[idx % palette.length];
        summaries.push({ name: tN, total: sub, color: color });

        let html = `<div class="group-card" onclick="showGroupDetails('${tN}')" style="border-left: 10px solid ${color}; padding:20px; border-radius:20px; box-shadow:0 5px 15px rgba(0,0,0,0.05); margin-bottom:15px; background:white; cursor:pointer; animation: fadeIn 0.5s ease;">
            <div class="group-header" style="display:flex; justify-content:space-between; font-size:18px; font-weight:normal; margin-bottom:15px; border-bottom:1px solid #eee; padding-bottom:10px; color:#444;">
                <span>${tN}</span><span>${sub.toLocaleString('en-IN')}</span>
            </div>`;

        Object.entries(vals).forEach(([iN, iV]) => {
            const p = sub > 0 ? Math.round((iV / sub) * 100) : 0;
            html += `<div style="margin-top:10px;">
                <div style="display:flex; justify-content:space-between; font-size:18px; margin-bottom:5px; color:#444;">
                    <span>${iN} - ${iV.toLocaleString('en-IN')}</span>
                    <span style="color:${color};">${p}%</span>
                </div>
                <div style="background:#f0f0f0; height:10px; border-radius:10px;">
                    <div style="width:${p}%; background:${color}; height:100%; border-radius:10px; transition: width 1s ease;"></div>
                </div>
            </div>`;
        });
        container.innerHTML += html + `</div>`; 
        idx++;
    });

    // সামারি কার্ড যেখানে গ্র্যান্ড টোটাল দেখাবে
    if (summaries.length > 0) {
        let kpiCardBody = `
            <div style="display:flex; justify-content:space-between; align-items:center; border-bottom: 2px solid #eee; margin-bottom: 15px; padding-bottom:10px;">
                <div style="display:flex; align-items:center; gap:8px;">
                    <span style="font-size:18px; font-weight:600; color:#333;">
                        ${currentLabel || " Summaries"}
                    </span>
                </div>
                <span id="dynamic-grand-total" style="font-size:18px; font-weight:700; color:#00bcd4;">0</span>
            </div>`;

        summaries.forEach((s, i) => {
            const isChecked = (savedSelections && savedSelections[s.name] === true); 
            const uniqueColor = palette[i % palette.length];
            
            kpiCardBody += `
            <div class="summary-row" data-name="${s.name}" data-value="${s.total}" style="margin-bottom:12px; opacity: ${isChecked ? '1' : '0.4'};">
                <div style="display:flex; justify-content:space-between; align-items:center; margin-bottom:5px;">
                    <div style="display:flex; align-items:center; gap:8px;">
                        <input type="checkbox" class="item-checkbox" ${isChecked ? 'checked' : ''} 
                            onchange="window.updateGrandTotal()" 
                            style="width:18px; height:18px; cursor:pointer; accent-color:${uniqueColor};">
                        <span style="font-size:18px; color:#444; font-weight:500;">${s.name} - ${s.total.toLocaleString('en-IN')}</span>
                    </div>
                    <span class="percentage-label" style="font-size:18px; font-weight:600; color:${uniqueColor};">0%</span>
                </div>
                <div style="background:#f0f0f0; height:10px; border-radius:10px; overflow:hidden;">
                    <div class="progress-bar-fill" style="width:0%; background:${uniqueColor}; height:100%; border-radius:10px; transition: width 0.5s ease;"></div>
                </div>
            </div>`;
        });

        container.innerHTML += `<div class="group-card" style="border-left: 12px solid #333; background: #fafafa; padding:25px; border-radius:20px; margin-bottom:15px; box-shadow:0 10px 20px rgba(0,0,0,0.1);">${kpiCardBody}</div>`;

        // রেন্ডার হওয়ার সাথে সাথে গ্র্যান্ড টোটাল রিফ্রেশ করা
        setTimeout(() => {
            if (typeof window.updateGrandTotal === 'function') {
                window.updateGrandTotal(false); 
            }
        }, 100);
    }
}

function renderAvgPie(achieved, balance) {
    const canvas = document.getElementById('avgPieChartCanvas');
    if (!canvas) return;
    const ctx = canvas.getContext('2d');
    if(window.avgPieChart) window.avgPieChart.destroy();
    window.avgPieChart = new Chart(ctx, { 
        type: 'pie', 
        data: { labels: ['Achieved', 'Balance'], datasets: [{ data: [achieved, balance], backgroundColor: ['#00bcd4', '#f44336'] }] }, 
        plugins: [ChartDataLabels], 
        options: { 
            responsive: true, maintainAspectRatio: false,
            plugins: { 
                legend: { display: false }, 
                datalabels: { 
                    display: true, color: '#fff', font: { size: 18, weight: 'normal' }, 
                    formatter: (v, ctx) => { 
                        const total = ctx.dataset.data[0] + ctx.dataset.data[1]; 
                        return total > 0 ? Math.round((v/total)*100)+'%' : ''; 
                    } 
                } 
            } 
        } 
    });
}
function renderChart(data) {
    const canvas = document.getElementById('productionChart');
    if (!canvas) return;
    const ctx = canvas.getContext('2d'); 
    if(window.myChart) window.myChart.destroy();

    // মোট ডাটার যোগফল বের করা
    const totalSum = data.reduce((a, b) => a + b, 0);

    window.myChart = new Chart(ctx, { 
        type: 'bar', 
        data: { labels: monthNames, datasets: [{ data, backgroundColor: palette, borderRadius: 10 }] }, 
        plugins: [ChartDataLabels], 
        options: { 
            layout: { padding: { top: 60 } }, 
            maintainAspectRatio: false, 
            plugins: { 
                legend: { display: false }, 
                datalabels: { 
                    anchor: 'end', 
                    align: 'top', 
                    offset: 5, 
                    // এখানে কালার ফাংশন ব্যবহার করা হয়েছে যাতে বারের কালার অনুযায়ী % দেখায়
                    color: (context) => {
                        return palette[context.dataIndex % palette.length];
                    }, 
                    font: { size: 18, weight: 'bold' }, 
                    formatter: (v) => {
                        if (v === 0) return ''; 
                        const percentage = totalSum > 0 ? Math.round((v / totalSum) * 100) : 0;
                        // ব্র্যাকেট () সরিয়ে দেওয়া হয়েছে
                        return `${v.toLocaleString('en-IN')}\n${percentage}%`;
                    },
                    textAlign: 'center'
                } 
            }, 
            scales: { 
                y: { display: false, grid: { display: false } }, 
                x: { grid: { display: false }, ticks: { font: { size: 14 } } }
            } 
        } 
    });
}
       // --- Analytics View (Updated for Shared Data) ---
window.loadAnalytics = async () => {
    if (!user) return;
    showSpinner(true);
    const selYear = document.getElementById('fYear').value;
    const tbody = document.getElementById('analyticsBody');
    tbody.innerHTML = ""; 
    let reportData = {};

    for (const sec of allSections) {
        // সরাসরি সেকশন মালিকের পাথ (sec.owner) থেকে ডেটা ফেচ করা হচ্ছে
        const dailySnap = await getDocs(collection(db, "user_data", sec.owner, "sections", sec.id, "daily_data"));
        
        dailySnap.forEach(doc => {
            if (doc.id.startsWith(selYear)) {
                const month = parseInt(doc.id.split('-')[1]);
                const data = doc.data();
                
                if (!reportData[sec.name]) {
                    reportData[sec.name] = { 
                        groups: {}, 
                        secTarget: 0, 
                        secTotalAchieved: 0, 
                        secMonthTotals: new Array(12).fill(0) 
                    };
                }
                
                // Target এবং Achieved সরাসরি Firebase-এর মেইন ফিল্ড থেকে নেওয়া হচ্ছে
                reportData[sec.name].secTarget += Number(data.Target) || 0;
                reportData[sec.name].secTotalAchieved += Number(data.Achieved) || 0; 

                const tables = data.tables || {};
                Object.entries(tables).forEach(([groupName, groupData]) => {
                    if (groupData.items) { 
                        if (!reportData[sec.name].groups[groupName]) {
                            reportData[sec.name].groups[groupName] = { 
                                items: {}, 
                                groupMonthTotals: new Array(12).fill(0) 
                            };
                        }
                        
                        Object.entries(groupData.items).forEach(([itemName, value]) => {
                            if (!reportData[sec.name].groups[groupName].items[itemName]) {
                                reportData[sec.name].groups[groupName].items[itemName] = { 
                                    months: new Array(12).fill(0) 
                                };
                            }
                            
                            let val = Number(value) || 0;
                            reportData[sec.name].groups[groupName].items[itemName].months[month - 1] += val;
                            reportData[sec.name].groups[groupName].groupMonthTotals[month - 1] += val;
                            reportData[sec.name].secMonthTotals[month - 1] += val;
                        });
                    }
                });
            }
        });
    }

    // Render Table Logic (আপনার দেওয়া স্টাইল অনুযায়ী)
    const bStyle = `border: 1px solid #000; padding: 6px 4px; text-align: center; font-size: 16px; font-weight: normal; vertical-align: middle; color: #000; background: none;`;

    Object.entries(reportData).forEach(([secName, secData]) => {
        const groups = Object.entries(secData.groups);
        let totalRows = 0;
        groups.forEach(([gn, gd]) => {
            const itemsCount = Object.keys(gd.items).length;
            totalRows += (itemsCount > 1) ? (itemsCount + 1) : itemsCount;
        });

        let isFirstSecRow = true;
        const target = secData.secTarget;
        const totalAch = secData.secTotalAchieved; 
        const perc = target > 0 ? Math.round((totalAch / target) * 100) : 0;

        groups.forEach(([groupName, groupData]) => {
            const items = Object.entries(groupData.items);
            let isFirstGrp = true;

            items.forEach(([itemName, itemData]) => {
                const tr = document.createElement('tr');
                let html = "";
                if (isFirstSecRow) html += `<td rowspan="${totalRows}" style="${bStyle}">${secName}</td>`;
                if (isFirstGrp) {
                    const gSpan = (items.length > 1) ? (items.length + 1) : items.length;
                    html += `<td rowspan="${gSpan}" style="${bStyle}">${groupName}</td>`;
                    isFirstGrp = false;
                }
                html += `<td style="${bStyle} text-align:left;">${itemName}</td>`;
                itemData.months.forEach(m => html += `<td style="${bStyle}">${m > 0 ? m : '-'}</td>`);
                
                const itemSum = itemData.months.reduce((a, b) => a + (Number(b) || 0), 0);
                html += `<td style="${bStyle}">${itemSum}</td>`;

                if (isFirstSecRow) {
                    html += `<td rowspan="${totalRows}" style="${bStyle}">${target}</td>`;
                    html += `<td rowspan="${totalRows}" style="${bStyle}">${totalAch}</td>`;
                    html += `<td rowspan="${totalRows}" style="${bStyle}">${perc}%</td>`;
                    isFirstSecRow = false;
                }
                tr.innerHTML = html;
                tbody.appendChild(tr);
            });

            if (items.length > 1) {
                const subTr = document.createElement('tr');
                let subHtml = `<td style="${bStyle} text-align:left;">${groupName} Total</td>`;
                groupData.groupMonthTotals.forEach(gm => subHtml += `<td style="${bStyle}">${gm > 0 ? gm : '-'}</td>`);
                const groupSum = groupData.groupMonthTotals.reduce((a, b) => a + b, 0);
                subHtml += `<td style="${bStyle}">${groupSum}</td>`;
                subTr.innerHTML = subHtml;
                tbody.appendChild(subTr);
            }
        });
    });
    showSpinner(false);
};

   
       // --- Data Entry Functions ---

window.fetchDataByDate = async () => {
    const date = document.getElementById('prodDate').value; 
    if(!date || !currentSecID) return;

    const section = allSections.find(s => s.id === currentSecID);
    if(section) {
        const labelT = document.getElementById('labelT_Entry');
        const labelA = document.getElementById('labelA_Entry');
        if(labelT) labelT.innerText = section.labelT || "Target";
        if(labelA) labelA.innerText = (section.labelA || "Achieved").toUpperCase();
    }

    // মাস্টার মোড প্যানেলটি ডিফল্টভাবে হাইড রাখা (যাতে ওপেন হয়ে না থাকে)
    const actions = document.getElementById('masterActions');
    if(actions) actions.classList.add('hidden');
    window.isMasterMode = false; // শুরুতে এটি false থাকবে

    try {
        const dailyDoc = await getDoc(doc(db, "user_data", section.owner, "sections", currentSecID, "daily_data", date));
        
        if(dailyDoc.exists()) {
            const d = dailyDoc.data(); 
            document.getElementById('tVal').value = d.Target || 0; 
            document.getElementById('aVal').value = d.Achieved || 0; 
            document.getElementById('prodNote').value = d.Note || "";
            renderTables(d.tables || {});
        } else {
            const secDoc = await getDoc(doc(db, "user_data", section.owner, "sections", currentSecID));
            const master = (secDoc.exists() ? secDoc.data().masterConfig : {}) || {};
            
            document.getElementById('tVal').value = 0; 
            document.getElementById('aVal').value = 0; 
            document.getElementById('prodNote').value = "";
            
            const resetTables = JSON.parse(JSON.stringify(master));
            Object.values(resetTables).forEach(t => { 
                if(t.items) Object.keys(t.items).forEach(k => t.items[k] = 0); 
            });
            renderTables(resetTables);
        }
    } catch (err) {
        console.error("Fetch Error:", err);
    }
};

function renderTables(tables) {
    const container = document.getElementById('dynamicContainer'); 
    container.innerHTML = "";
    
    Object.entries(tables).forEach(([tN, tD]) => {
        const div = document.createElement('div'); 
        div.className = "table-block"; 
        div.style.cssText = "border:1px solid #ddd; padding:10px; border-radius:10px; margin-bottom:10px; background:#fafafa;";
        
        let subtotal = Object.values(tD.items || {}).reduce((acc, val) => acc + (parseFloat(val) || 0), 0);
        
        div.innerHTML = `
            <div style="display:flex; justify-content:space-between; align-items:center;">
                <div style="display:flex; align-items:center;">
                    ${isMasterMode ? `<button onclick="deleteTable('${tN}')" style="background:none; border:none; color:red; font-weight:bold; cursor:pointer; margin-right:8px;">✕</button>` : ''}
                    <strong class="table-name" contenteditable="${isMasterMode}" style="outline:none; cursor:${isMasterMode ? 'text' : 'default'}">${tN}</strong>
                </div>
                <div>
                    <span class="subtotal-display" id="subtotal-${tN}" style="margin-right:10px; font-weight:bold;">${subtotal}</span>
                    <input type="checkbox" ${tD.selected ? 'checked' : ''} 
                           onchange="calcTotal()" class="table-check" data-name="${tN}">
                </div>
            </div>
            <div id="items-${tN}">
                ${Object.entries(tD.items || {}).map(([iN, iV]) => `
                    <div class="item-row" style="display:flex; justify-content:space-between; align-items:center; margin-top:5px;">
                        <div style="display:flex; align-items:center;">
                            ${isMasterMode ? `<button onclick="removeItem('${tN}', '${iN}')" style="background:none; border:none; color:red; font-size:12px; cursor:pointer; margin-right:5px;">✕</button>` : ''}
                            <span class="item-name" contenteditable="${isMasterMode}" style="outline:none; cursor:${isMasterMode ? 'text' : 'default'}">${iN}</span>
                        </div>
                        <input type="number" value="${iV}" class="val-in item-input" 
                               data-table="${tN}" data-item="${iN}" oninput="calcTotal()">
                    </div>
                `).join('')}
            </div>
            ${isMasterMode ? `<button onclick="addItem('${tN}')" style="margin-top:8px; font-size:11px; color:blue; background:none; border:none; cursor:pointer;">+ Add Item</button>` : ''}
        `;
        container.appendChild(div);
    });
    calcTotal();
}

window.calcTotal = () => {
    let grandTotal = 0;
    
    document.querySelectorAll('.table-block').forEach(block => {
        let tableSub = 0;
        const isChecked = block.querySelector('.table-check').checked;
        
        block.querySelectorAll('.item-input').forEach(input => {
            const val = Number(input.value) || 0;
            tableSub += val;
        });

        const subDisplay = block.querySelector('.subtotal-display');
        if (subDisplay) subDisplay.innerText = tableSub;
        
        if (isChecked) grandTotal += tableSub;
    });

    const aValInput = document.getElementById('aVal');
    if (aValInput) aValInput.value = grandTotal;
};

// --- Management & Saving Logic ---

function getCurrentTables() { 
    const tables = {}; 
    document.querySelectorAll('.table-block').forEach(block => { 
        const tableName = block.querySelector('.table-name').innerText.trim();
        const isChecked = block.querySelector('.table-check').checked;
        
        tables[tableName] = { selected: isChecked, items: {} }; 
        
        block.querySelectorAll('.item-row').forEach(row => { 
            const itemName = row.querySelector('.item-name').innerText.trim();
            const itemVal = Number(row.querySelector('.item-input').value) || 0;
            tables[tableName].items[itemName] = itemVal;
        });
    });
    return tables;
}

window.saveData = async () => {
    const date = document.getElementById('prodDate').value; 
    if(!date) return alert("Please select a date");
    
    if (typeof showSpinner === 'function') showSpinner(true);
    
    const tables = getCurrentTables(); 
    const payload = { 
        Target: Number(document.getElementById('tVal').value) || 0, 
        Achieved: Number(document.getElementById('aVal').value) || 0, 
        Note: document.getElementById('prodNote').value || "", 
        tables: tables 
    };

    try {
        await setDoc(doc(db, "user_data", currentSecOwner, "sections", currentSecID, "daily_data", date), payload);
        
        if(typeof isMasterMode !== 'undefined' && isMasterMode) {
            await updateDoc(doc(db, "user_data", currentSecOwner, "sections", currentSecID), { 
                masterConfig: tables 
            });
        }
        
        if (typeof closeModal === 'function') closeModal('entryModal');

        await Swal.fire({
            title: 'Saved!',
            text: 'Everything has been saved successfully.',
            icon: 'success',
            confirmButtonColor: '#00bcd4',
            timer: 2000
        });

        if(typeof updatePresentation === 'function') updatePresentation();
        if(typeof loadDashboard === 'function') loadDashboard();

    } catch (err) { 
        console.error("Save Error:", err);
        Swal.fire({ title: 'Save Failed', text: err.message, icon: 'error' });
    } finally {
        if (typeof showSpinner === 'function') showSpinner(false);
    }
};

// --- মাস্টার মোড ফাংশন (ক্লিক করলে ওপেন হবে) ---

window.toggleMasterMode = () => {
    // বর্তমান অবস্থার বিপরীত করা
    isMasterMode = !isMasterMode;
    
    const actions = document.getElementById('masterActions');
    if(actions) {
        if(isMasterMode) {
            actions.classList.remove('hidden'); // শো করবে
        } else {
            actions.classList.add('hidden'); // হাইড করবে
        }
    }
    
    // ভিউ আপডেট করার জন্য টেবিলগুলো পুনরায় রেন্ডার করা
    const current = getCurrentTables();
    renderTables(current);
};

window.deleteTable = async (tN) => {
    const result = await Swal.fire({
        title: 'Are you sure?',
        text: `Delete '${tN}'?`,
        icon: 'warning',
        showCancelButton: true,
        confirmButtonColor: '#d33',
        cancelButtonColor: '#3085d6'
    });
    if (result.isConfirmed) {
        const current = getCurrentTables();
        if (current && current[tN]) {
            delete current[tN];
            renderTables(current);
        }
    }
};

window.removeItem = async (tN, iN) => {
    const result = await Swal.fire({
        title: 'Remove Item?',
        text: `Delete '${iN}'?`,
        icon: 'warning',
        showCancelButton: true,
        confirmButtonColor: '#d33',
        cancelButtonColor: '#3085d6'
    });
    if (result.isConfirmed) {
        const current = getCurrentTables();
        if (current && current[tN]?.items) {
            delete current[tN].items[iN];
            renderTables(current);
        }
    }
};

window.addTable = async () => {
    const { value: name } = await Swal.fire({
        title: 'Enter Table Name',
        input: 'text',
        showCancelButton: true,
        inputPlaceholder: 'e.g., Line 01, Shift A',
        inputValidator: (v) => !v && 'Name required!',
        confirmButtonColor: '#00bcd4'
    });
    
    if (name) {
        const current = getCurrentTables(); 
        const exists = Object.keys(current).some(key => key.toLowerCase() === name.toLowerCase());
        if (exists) return Swal.fire('Error', 'Table already exists!', 'error');

        current[name] = { selected: true, items: {} }; 
        renderTables(current);
    }
};

window.addItem = async (tN) => {
    const { value: name } = await Swal.fire({
        title: `Add to ${tN}`,
        input: 'text',
        showCancelButton: true,
        inputPlaceholder: 'Item Name',
        inputValidator: (v) => !v && 'Name required!',
        confirmButtonColor: '#00bcd4'
    });

    if (name) {
        const current = getCurrentTables(); 
        if(current[tN]) { 
            if (!current[tN].items[name]) {
                current[tN].items[name] = 0; 
                renderTables(current); 
            } else {
                Swal.fire('Error', 'Item already exists in this table!', 'error');
            }
        }
    }
};
window.createSection = async () => {
    const inputName = document.getElementById('newSecName');
    const name = inputName?.value.trim() || "";
    
    // ইনপুট থেকে কাস্টম লেবেল নেওয়া
    const lT = document.getElementById('labelT')?.value.trim() || "Target";
    const lA = document.getElementById('labelA')?.value.trim() || "Achieved";
    const lB = document.getElementById('labelB')?.value.trim() || "Balance";

    if (!name) {
        return Swal.fire({
            title: 'নাম প্রয়োজন',
            text: 'দয়া করে সেকশনের একটি নাম দিন!',
            icon: 'info',
            confirmButtonColor: '#00bcd4'
        });
    }

    // --- ডুপ্লিকেট চেক (ক্রাশ প্রোটেকশন সহ) ---
    const checkLength = 4; 
    const newNamePart = name.substring(0, Math.min(name.length, checkLength)).toLowerCase();
    
    const isDuplicate = (allSections || []).some(sec => {
        // sec.sectionName অথবা sec.name যেটাই থাকুক সেটা নিবে
        const existingName = sec.sectionName || sec.name || ""; 
        if (!existingName) return false;
        
        const existingNamePart = existingName.substring(0, Math.min(existingName.length, checkLength)).toLowerCase();
        return existingNamePart === newNamePart;
    });

    if (isDuplicate) {
        return Swal.fire({
            title: 'Duplicate Detected!',
            text: `A section with similar starting characters ("${newNamePart}") already exists.`,
            icon: 'warning',
            confirmButtonColor: '#00bcd4'
        });
    }

    if (typeof showSpinner === 'function') showSpinner(true);

    try {
        // ফায়ারবেস ম্যাপে ডট (.) সাপোর্ট করে না, তাই ইমেইল কি ফরম্যাট করা
        const emailKey = user.email.replace(/\./g, '_dot_');

        // --- ডাটাবেস স্ট্রাকচার (Array + Roles + Labels) ---
        const newSectionData = {
            sectionName: name,           // ড্যাশবোর্ডের জন্য
            name: name,                  // ব্যাকওয়ার্ড সাপোর্ট
            ownerEmail: user.email,      // মালিক
            createdAt: new Date().toISOString(),
            sharedWith: [user.email],    // Array format
            roles: {                     // Roles Map
                [emailKey]: "admin"      // ডট রিপ্লেস করা ইমেইল কি হিসেবে ব্যবহার
            },
            labelT: lT,                  // Target
            labelA: lA,                  // Achieved
            labelB: lB,                  // Balance
            deleted: false,
            showContribution: true 
        };

        // ডাটাবেসে সেভ করা
        const userDocRef = collection(db, "user_data", user.email, "sections");
        await addDoc(userDocRef, newSectionData);
        
        // ক্লিনআপ
        if (inputName) inputName.value = "";
        if (typeof closeModal === 'function') closeModal('createModal');
        
        await Swal.fire({ 
            title: 'Success!', 
            text: 'Section created successfully.', 
            icon: 'success',
            confirmButtonColor: '#00bcd4'
        });

        // ড্যাশবোর্ড রিফ্রেশ
        if (typeof window.loadDashboard === 'function') await window.loadDashboard();
        else if (typeof loadSections === 'function') loadSections(); 
        else location.reload();

    } catch (err) {
        console.error("Create Error:", err);
        // পারমিশন এরর হলে ইউজারকে জানানো
        Swal.fire({ 
            title: 'Permission Denied!', 
            text: 'Could not create section. Please check your Firestore rules.', 
            icon: 'error',
            confirmButtonColor: '#f44336'
        });
    } finally {
        if (typeof showSpinner === 'function') showSpinner(false);
    }
};
window.openShareModal = (e, id, name) => {
    e.stopPropagation(); 
    window.currentSecID = id; 
    const modalTitle = document.getElementById('sharingSecName');
    if(modalTitle) modalTitle.innerText = `Share: ${name}`;
    
    const list = document.getElementById('shareList');
    if(list) list.innerHTML = "<p style='text-align:center; padding:20px;'>Loading permissions...</p>"; 
    
    openModal('shareModal'); 
    loadPermissions(id);
};

async function loadPermissions(id) {
    try {
        // ১. সেকশনটি খুঁজে বের করা যাতে ওনারের ইমেইল পাওয়া যায়
        const section = (typeof allSections !== 'undefined') ? allSections.find(s => s.id === id) : null;
        const ownerEmail = (section && section.owner) ? section.owner : (user ? user.email : null);
        
        if (!ownerEmail) throw new Error("Owner not found");

        // ২. সঠিক ওনারের পাথ থেকে ডকুমেন্ট গেট করা
        const secDoc = await getDoc(doc(db, "user_data", ownerEmail, "sections", id));
        if (!secDoc.exists()) throw new Error("Section document missing");

        const data = secDoc.data();
        const sharedWith = data.sharedWith || []; 
        const roles = data.roles || {};           
        
        const list = document.getElementById('shareList');
        if(!list) return;

        list.innerHTML = "<h4 style='margin-bottom:10px; border-bottom:1px solid #ddd; padding-bottom:5px;'>Shared with:</h4>";

        if (sharedWith.length === 0) {
            list.innerHTML += "<p style='color:#888; font-size:14px; text-align:center; padding:10px;'>No users added yet.</p>";
            return;
        }

        sharedWith.forEach((email) => {
            const emailKey = email.replace(/\./g, '_dot_');
            const role = roles[emailKey] || 'viewer';
            
            list.innerHTML += `
                <div class="share-item" style="display:flex; justify-content:space-between; align-items:center; padding:10px 0; border-bottom:1px solid #eee;">
                    <div>
                        <div style="font-weight:bold; font-size:14px; color:#333;">${email}</div>
                        <div style="font-size:12px; color:var(--primary, #00bcd4); text-transform:capitalize;">${role}</div>
                    </div>
                    <button onclick="removePermission('${email}')" 
                            style="color:#ff5252; border:1px solid #ff5252; background:none; padding:5px 12px; border-radius:4px; cursor:pointer; font-size:12px; transition:0.3s;">
                        Remove
                    </button>
                </div>`;
        });
    } catch (err) {
        console.error("Load Permissions Error:", err);
        const list = document.getElementById('shareList');
        if(list) list.innerHTML = `<p style="color:red; font-size:12px;">Error loading permissions: ${err.message}</p>`;
    }
}

window.addPermission = async () => {
    const emailInput = document.getElementById('shareEmail');
    const roleInput = document.getElementById('shareRole');
    if (!emailInput || !roleInput) return;

    const email = emailInput.value.toLowerCase().trim();
    const role = roleInput.value;

    if (!email || !email.includes('@')) {
        return Swal.fire('Error', 'Please enter a valid email address.', 'warning');
    }

    if (typeof showSpinner === 'function') showSpinner(true);

    try {
        const sid = window.currentSecID || (typeof persistentSectionID !== 'undefined' ? persistentSectionID : null);
        if (!sid) throw new Error("No active section selected.");

        const section = (typeof allSections !== 'undefined') ? allSections.find(s => s.id === sid) : null;
        const ownerEmail = (section && section.owner) ? section.owner : (user ? user.email : null);
        
        if (!ownerEmail) throw new Error("Could not determine section owner.");

        const secRef = doc(db, "user_data", ownerEmail, "sections", sid);
        const emailKey = email.replace(/\./g, '_dot_');

        await updateDoc(secRef, { 
            sharedWith: arrayUnion(email),
            [`roles.${emailKey}`]: role 
        });

        emailInput.value = "";
        await loadPermissions(sid); 
        
        Swal.fire({
            title: 'Success',
            text: `${email} added as ${role}`,
            icon: 'success',
            confirmButtonColor: '#00bcd4'
        });

    } catch (err) {
        console.error("Add Permission Error:", err);
        Swal.fire('Failed', err.message, 'error');
    } finally {
        if (typeof showSpinner === 'function') showSpinner(false);
    }
};

window.removePermission = async (targetEmail) => {
    const result = await Swal.fire({
        title: 'Are you sure?',
        text: `Revoke all access for ${targetEmail}?`,
        icon: 'warning',
        showCancelButton: true,
        confirmButtonColor: '#d33',
        confirmButtonText: 'Yes, remove!'
    });

    if (!result.isConfirmed) return;
    if (typeof showSpinner === 'function') showSpinner(true);

    try {
        const sid = window.currentSecID || (typeof persistentSectionID !== 'undefined' ? persistentSectionID : null);
        const section = (typeof allSections !== 'undefined') ? allSections.find(s => s.id === sid) : null;
        const ownerEmail = (section && section.owner) ? section.owner : (user ? user.email : null);

        if (!ownerEmail) throw new Error("Owner info missing.");

        const secRef = doc(db, "user_data", ownerEmail, "sections", sid);
        const emailKey = targetEmail.replace(/\./g, '_dot_');

        await updateDoc(secRef, { 
            sharedWith: arrayRemove(targetEmail),
            [`roles.${emailKey}`]: deleteField()
        });
        
        await loadPermissions(sid);
        Swal.fire('Removed', 'Access revoked successfully.', 'success');
    } catch (err) {
        console.error("Remove Permission Error:", err);
        Swal.fire('Error', err.message, 'error');
    } finally {
        if (typeof showSpinner === 'function') showSpinner(false);
    }
};
// --- Updated JavaScript Functions (14px Font, Compact Excel Style) ---
window.showGroupDetails = async (groupName) => {
    if (typeof showSpinner === 'function') showSpinner(true);
    
    const sid = document.getElementById('fSecFilter').value || (typeof persistentSectionID !== 'undefined' ? persistentSectionID : '');
    const year = document.getElementById('fYear').value;
    const month = selectedMonth.toString().padStart(2, '0');
    const prefix = `${year}-${month}`;

    try {
        const section = allSections.find(s => s.id === sid);
        if (!section) return;

        const dailySnap = await getDocs(collection(db, "user_data", section.owner, "sections", sid, "daily_data"));

        let dailyDataMap = {};
        let itemNames = new Set();
        let updatedDates = [];

        dailySnap.forEach(doc => {
            if (doc.id.startsWith(prefix)) {
                const data = doc.data();
                const tableData = data.tables && data.tables[groupName];
                if (tableData && tableData.items) {
                    dailyDataMap[doc.id] = tableData.items;
                    updatedDates.push(doc.id);
                    Object.keys(tableData.items).forEach(name => itemNames.add(name));
                }
            }
        });

        updatedDates.sort();
        const itemsArray = Array.from(itemNames);

        // ১. হেডার
        let headHtml = `<tr style="font-size: 14px;">
            <th style="border: 1px solid #000; padding: 4px; font-weight: bold;">Date</th>`;
        itemsArray.forEach(name => { 
            headHtml += `<th style="border: 1px solid #000; padding: 4px; font-weight: bold;">${name}</th>`; 
        });
        headHtml += `<th style="border: 1px solid #000; padding: 4px; font-weight: bold;">Total</th></tr>`;
        document.getElementById('dynamicHeadings').innerHTML = headHtml;

        // ২. বডি
        let bodyHtml = "";
        let columnTotals = new Array(itemsArray.length).fill(0);
        let grandTotal = 0;

        updatedDates.forEach(dateKey => {
            const items = dailyDataMap[dateKey];
            let rowTotal = 0;
            const p = dateKey.split('-');
            const displayDate = `${p[2]}-${p[1]}-${p[0]}`;

            bodyHtml += `<tr style="font-size: 14px; text-align: center; line-height: 1.1;">
                <td style="border: 1px solid #000; padding: 2px;">${displayDate}</td>`; 
            
            itemsArray.forEach((name, idx) => {
                const val = Number(items[name]) || 0;
                bodyHtml += `<td style="border: 1px solid #000; padding: 2px;">${val > 0 ? val.toLocaleString() : '-'}</td>`;
                columnTotals[idx] += val;
                rowTotal += val;
            });
            bodyHtml += `<td style="border: 1px solid #000; padding: 2px;">${rowTotal.toLocaleString()}</td></tr>`;
            grandTotal += rowTotal;
        });
        document.getElementById('groupDetailBody').innerHTML = bodyHtml;

        // ৩. ফুটার
        let footHtml = `<tr style="font-weight:bold; font-size: 14px; text-align: center;">
            <td style="border: 1px solid #000; padding: 4px;">Total</td>`;
        columnTotals.forEach(t => { 
            footHtml += `<td style="border: 1px solid #000; padding: 4px;">${t.toLocaleString()}</td>`; 
        });
        footHtml += `<td style="border: 1px solid #000; padding: 4px;">${grandTotal.toLocaleString()}</td></tr>`;
        document.getElementById('groupDetailFoot').innerHTML = footHtml;

        // ৪. স্টাইলিং ও কন্টেইনার ফিক্স
        const modal = document.getElementById('groupDetailsModal');
        const modalBody = modal.querySelector('.modal-body') || modal.querySelector('.a4-page').parentElement;

        if (modalBody) {
            modalBody.style.maxHeight = "80vh";
            modalBody.style.overflowY = "auto";
        }
        
        const tableElement = document.getElementById('reportTableExcel');
        if(tableElement) {
            tableElement.style.borderCollapse = "collapse";
            tableElement.style.width = "100%";
        }

        // ৫. নাম আপডেট (এখানে আপনার HTML এর reportTitle আইডি ব্যবহার করা হয়েছে)
        if(section) {
            document.getElementById('reportTitle').innerText = `${section.name} - ${groupName}`;
        }

    } catch (error) {
        console.error("Error:", error);
    }

    if (typeof showSpinner === 'function') showSpinner(false);
    openModal('groupDetailsModal');
};


       window.showOverviewDetails = async () => {
    showSpinner(true); 
    try {
        const sid = document.getElementById('fSecFilter').value || (typeof persistentSectionID !== 'undefined' ? persistentSectionID : null);
        const year = document.getElementById('fYear').value;
        const monthEnabled = document.getElementById('monthEnable').checked;
        const prefix = monthEnabled ? `${year}-${selectedMonth.toString().padStart(2,'0')}` : `${year}`;

        const section = allSections.find(s => s.id === sid);
        if(!section) { showSpinner(false); return; }

        const lT = section.labelT || "Target";
        const lA = section.labelA || "Achieved";

        // ১. হেডার (ফন্ট সাইজ ১৪ এবং কালো বর্ডার)
        const tableHeader = document.querySelector('#overviewDetailsModal table thead');
        if (tableHeader) {
            tableHeader.innerHTML = `
                <tr style="font-size: 14px; background: transparent;">
                    <th style="border: 1px solid #000; padding: 4px;">Date</th>
                    <th style="border: 1px solid #000; padding: 4px;">${lT}</th>
                    <th style="border: 1px solid #000; padding: 4px;">${lA}</th>
                    <th style="border: 1px solid #000; padding: 4px;">Achi%</th>
                    <th style="border: 1px solid #000; padding: 4px;">Note</th>
                </tr>`;
        }

        const dailySnap = await getDocs(collection(db, "user_data", section.owner, "sections", sid, "daily_data"));
        
        let bodyHtml = "", totalT = 0, totalA = 0, dataList = [];
        dailySnap.forEach(ds => {
            if(ds.id.startsWith(prefix)) dataList.push({ id: ds.id, ...ds.data() });
        });
        dataList.sort((a, b) => a.id.localeCompare(b.id));

        dataList.forEach(d => {
            const t = Number(d.Target) || 0;
            const a = Number(d.Achieved) || 0;
            
            // পারসেন্টেজ রাউন্ড করা হয়েছে (দশমিক ছাড়া)
            const percent = t > 0 ? Math.round((a / t) * 100) : 0;
            
            const dateParts = d.id.split('-');
            const fDate = dateParts.length === 3 ? `${dateParts[2]}-${dateParts[1]}-${dateParts[0]}` : d.id;
            
            totalT += t; totalA += a;
            
            // বডি ডাটা (ফন্ট সাইজ ১৪ এবং কমপ্যাক্ট প্যাডিং)
            bodyHtml += `
                <tr style="text-align: center; font-size: 14px; line-height: 1.1;">
                    <td style="border: 1px solid #000; padding: 3px;">${fDate}</td>
                    <td style="border: 1px solid #000; padding: 3px;">${t.toLocaleString()}</td>
                    <td style="border: 1px solid #000; padding: 3px;">${a.toLocaleString()}</td>
                    <td style="border: 1px solid #000; padding: 3px;">${percent}%</td>
                    <td style="border: 1px solid #000; padding: 3px; text-align: left;">${d.Note || ""}</td>
                </tr>`;
        });

        document.getElementById('ovDetailBody').innerHTML = bodyHtml;
        
        const totalP = totalT > 0 ? Math.round((totalA / totalT) * 100) : 0;
        
        // ২. ফুটার (ফন্ট সাইজ ১৪)
        document.getElementById('ovDetailFoot').innerHTML = `
            <tr style="font-weight: bold; text-align: center; font-size: 14px;">
                <td style="border: 1px solid #000; padding: 5px;">TOTAL</td>
                <td style="border: 1px solid #000; padding: 5px;">${totalT.toLocaleString()}</td>
                <td style="border: 1px solid #000; padding: 5px;">${totalA.toLocaleString()}</td>
                <td style="border: 1px solid #000; padding: 5px;">${totalP}%</td>
                <td style="border: 1px solid #000; padding: 5px;">-</td>
            </tr>`;

        // ৩. টেবিল ও মডাল কন্টেইনার সেটিংস
        const modal = document.getElementById('overviewDetailsModal');
        const tableContainer = modal.querySelector('.modal-body'); 
        if(tableContainer) {
            tableContainer.style.maxHeight = "75vh";
            tableContainer.style.overflowY = "auto";
            tableContainer.style.padding = "0";
        }
        
        const tableElement = modal.querySelector('table');
        if(tableElement) {
            tableElement.style.borderCollapse = "collapse";
            tableElement.style.width = "100%";
        }

        if(section) document.getElementById('rpSection').innerText = section.name;
        openModal('overviewDetailsModal');
    } catch (e) {
        console.error("Overview Modal Error:", e);
    } finally {
        showSpinner(false);
    }
};
window.downloadExcel = () => {
    // ১. বর্তমানে কোন মডাল বা ভিউ ওপেন আছে সেটা খুঁজে বের করা
    let activeArea = null;
    let fileName = "Production_Report";

    if (document.getElementById('overviewDetailsModal').style.display === 'block') {
        activeArea = document.getElementById('overviewDetailsModal');
        fileName = "Overview_Report";
    } else if (document.getElementById('groupDetailsModal').style.display === 'block') {
        activeArea = document.getElementById('groupDetailsModal');
        fileName = "Group_Details_Report";
    } else if (!document.getElementById('analyticsView').classList.contains('hidden')) {
        activeArea = document.getElementById('analyticsView');
        fileName = "Annual_Analytics_Report";
    }

    if (!activeArea) {
        alert("কোন ডাটা পাওয়া যায়নি!");
        return;
    }

    // ২. ওপেন থাকা এরিয়া থেকে ডাটা এবং টেবিল কালেক্ট করা
    const table = activeArea.querySelector('table');
    if (!table) return alert("টেবিলে কোনো ডাটা নেই!");

    const coName = activeArea.querySelector('h1')?.innerText || "SMART PRODUCTION ERP";
    const address = activeArea.querySelector('p')?.innerText || "Production Management System";
    const reportTitle = activeArea.querySelector('h2, h3')?.innerText || "Report Details";
    
    // ডাইনামিক কলাম সংখ্যা বের করা (টাইটেল সেন্টারিং এর জন্য)
    const firstRow = table.querySelector('tr');
    const colCount = firstRow ? firstRow.cells.length : 10;

    const tableHTML = table.innerHTML;

    // ৩. এক্সেল টেমপ্লেট তৈরি (উন্নত স্টাইলসহ)
    const excelTemplate = `
        <html xmlns:o="urn:schemas-microsoft-com:office:office" xmlns:x="urn:schemas-microsoft-com:office:excel" xmlns="http://www.w3.org/TR/REC-html40">
        <head>
            <meta charset="UTF-8">
            <style>
                table { border-collapse: collapse; }
                th, td { border: 0.5pt solid #000000; text-align: center; padding: 5px; font-family: Calibri, Arial; font-size: 11pt; }
                .header-title { font-size: 18pt; font-weight: bold; border: none; }
                .header-sub { font-size: 11pt; border: none; color: #444; }
                .report-name { font-size: 14pt; font-weight: bold; border: none; text-decoration: underline; padding-bottom: 20px; }
                thead tr { background-color: #f2f2f2; font-weight: bold; }
                tfoot tr { background-color: #d9e9ff; font-weight: bold; }
            </style>
        </head>
        <body>
            <table>
                <tr><th colspan="${colCount}" class="header-title">${coName}</th></tr>
                <tr><td colspan="${colCount}" class="header-sub">${address}</td></tr>
                <tr><th colspan="${colCount}" class="report-name">${reportTitle}</th></tr>
                <tr></tr> ${tableHTML}
            </table>
        </body>
        </html>
    `;

    // ৪. ফাইল ডাউনলোড প্রসেস
    try {
        const blob = new Blob([excelTemplate], { type: 'application/vnd.ms-excel' });
        const url = URL.createObjectURL(blob);
        const link = document.createElement("a");
        link.href = url;
        link.download = `${fileName}_${new Date().getTime()}.xls`;
        document.body.appendChild(link);
        link.click();
        document.body.removeChild(link);
        URL.revokeObjectURL(url);
    } catch (err) {
        console.error("Export Error:", err);
        alert("এক্সেল ফাইল তৈরি করতে সমস্যা হয়েছে।");
    }
};

function printReport(type) {
    const body = document.body;
    
    // আগের মোড রিমুভ করা
    body.classList.remove('landscape-mode', 'portrait-mode');
    
    // টাইপ অনুযায়ী মোড সেট করা
    if (type === 'analytics') {
        body.classList.add('landscape-mode');
    } else {
        body.classList.add('portrait-mode');
    }
    
    // প্রিন্ট কমান্ড
    window.print();
}

</script>
</body>
</html>