import os
import json
import streamlit as st
from PIL import Image

# ----------------------------------------------------
# Configuration & Storage Setup
# ----------------------------------------------------
st.set_page_config(
    page_title="Hridoy Study Hub",
    page_icon="📚",
    layout="wide",
    initial_sidebar_state="expanded"
)

DATA_DIR = "app_data"
UPLOADS_DIR = os.path.join(DATA_DIR, "uploads")
META_FILE = os.path.join(DATA_DIR, "metadata.json")

os.makedirs(UPLOADS_DIR, exist_ok=True)

# Complete syllabus structure for all requested sections
SUBJECT_STRUCTURE = {
    "Computer Science": [
        "General IT & Architecture",
        "Data Structures & Algorithms",
        "DBMS & SQL",
        "Computer Networks",
        "Operating Systems",
        "Programming Concepts"
    ],
    "Mathematics": [
        "Number System",
        "Arithmetic",
        "Algebra",
        "Geometry & Mensuration",
        "Trigonometry",
        "Statistics & Probability"
    ],
    "English": [
        "Grammar & Usage",
        "Vocabulary & Idioms",
        "Comprehension",
        "Sentence Correction"
    ],
    "Reasoning": [
        "Verbal Reasoning",
        "Non-Verbal Reasoning",
        "Analytical Reasoning",
        "Logical Puzzles"
    ],
    "General Science": [
        "Physics",
        "Chemistry",
        "Biology",
        "Environmental Science"
    ],
    "General Knowledge": {
        "History": [
            "Indian History (Ancient, Medieval, Modern)",
            "World History",
            "Assam History"
        ],
        "Geography": [
            "Indian Geography",
            "Physical & World Geography",
            "Assam Geography"
        ],
        "Civics & Current Affairs": [
            "Indian Polity & Constitution",
            "National Affairs",
            "State Affairs (Assam)",
            "Monthly Current Affairs"
        ]
    }
}

def load_data():
    if os.path.exists(META_FILE):
        try:
            with open(META_FILE, "r", encoding="utf-8") as f:
                return json.load(f)
        except Exception:
            return {}
    return {}

def save_data(data):
    with open(META_FILE, "w", encoding="utf-8") as f:
        json.dump(data, f, indent=4)

if "db" not in st.session_state:
    st.session_state.db = load_data()

# ----------------------------------------------------
# Sidebar Navigation
# ----------------------------------------------------
st.sidebar.title("📚 Hridoy Study Hub")
st.sidebar.caption("Exam Preparation & Notes Portal")
st.sidebar.markdown("---")

main_subject = st.sidebar.selectbox("Select Subject", list(SUBJECT_STRUCTURE.keys()))

if main_subject == "General Knowledge":
    gk_subsection = st.sidebar.selectbox("GK Sub-Section", list(SUBJECT_STRUCTURE["General Knowledge"].keys()))
    topic = st.sidebar.selectbox("Select Topic", SUBJECT_STRUCTURE["General Knowledge"][gk_subsection])
    key_path = f"GK_{gk_subsection}_{topic}"
else:
    topic = st.sidebar.selectbox("Select Topic", SUBJECT_STRUCTURE[main_subject])
    key_path = f"{main_subject}_{topic}"

# Ensure data entry exists for the selected topic
if key_path not in st.session_state.db:
    st.session_state.db[key_path] = {
        "completed": False,
        "completion_notes": "",
        "notes_text": "",
        "images": [],
        "exercises": [],
        "qa_list": []
    }

topic_data = st.session_state.db[key_path]

# ----------------------------------------------------
# Header & Syllabus Completion Status
# ----------------------------------------------------
st.title("🎯 Hridoy Study Hub")
st.subheader(f"📖 {main_subject} → {topic}")

col1, col2 = st.columns([1, 2])
with col1:
    is_done = st.checkbox("Mark Topic as Syllabus Completed", value=topic_data.get("completed", False))
    if is_done != topic_data.get("completed", False):
        topic_data["completed"] = is_done
        save_data(st.session_state.db)
        st.success("Syllabus status updated!")

with col2:
    status_note = st.text_input(
        "Completion Milestone / Remarks", 
        value=topic_data.get("completion_notes", ""), 
        placeholder="e.g., Theory done, Revision pending"
    )
    if status_note != topic_data.get("completion_notes", ""):
        topic_data["completion_notes"] = status_note
        save_data(st.session_state.db)

st.markdown("---")

# ----------------------------------------------------
# Main Tabs: Slide Viewer, Uploads, Exercises, Notes, Q&A
# ----------------------------------------------------
tab_slides, tab_upload, tab_exercises, tab_notes, tab_qa = st.tabs([
    "🖼️ Visual Notes & Slider", 
    "📤 Upload Pictures", 
    "📑 Exercise Ranges", 
    "📝 Theory Notes", 
    "❓ Q&A Practice Sets"
])

# 1. Visual Notes & Shuffle Slider
with tab_slides:
    images = topic_data.get("images", [])
    if images:
        col_ctrl1, col_ctrl2 = st.columns([3, 1])
        with col_ctrl1:
            slide_idx = st.slider("Quickly shuffle through notes:", 1, len(images), 1) - 1
        with col_ctrl2:
            st.metric("Total Slides", len(images), f"Current: Slide {slide_idx + 1}")
        
        img_path = images[slide_idx]
        if os.path.exists(img_path):
            img = Image.open(img_path)
            st.image(img, caption=f"Slide {slide_idx + 1} of {len(images)}: {os.path.basename(img_path)}", use_container_width=True)
        else:
            st.error("Image file not found on disk.")
    else:
        st.info("No visual notes uploaded for this topic yet. Go to the 'Upload Pictures' tab to add pages.")

# 2. Sequential Upload Tab
with tab_upload:
    st.subheader("Upload Sequenced Notes (Images)")
    uploaded_files = st.file_uploader(
        "Upload slide photos (PNG, JPG, JPEG)", 
        type=["png", "jpg", "jpeg"], 
        accept_multiple_files=True
    )
    
    if st.button("Save Uploaded Images in Sequence"):
        if uploaded_files:
            topic_folder = os.path.join(UPLOADS_DIR, key_path.replace(" ", "_").replace("/", "_"))
            os.makedirs(topic_folder, exist_ok=True)
            
            saved_paths = []
            for file in uploaded_files:
                file_path = os.path.join(topic_folder, file.name)
                with open(file_path, "wb") as f:
                    f.write(file.getbuffer())
                saved_paths.append(file_path)
            
            topic_data["images"].extend(saved_paths)
            save_data(st.session_state.db)
            st.success(f"Added {len(saved_paths)} image(s) to this topic sequence!")
            st.rerun()

    if topic_data.get("images"):
        if st.button("Clear All Slides For This Topic", type="secondary"):
            topic_data["images"] = []
            save_data(st.session_state.db)
            st.rerun()

# 3. Exercise & Slide Range Mapping
with tab_exercises:
    st.subheader("Exercise & Slide Range Mapping")
    with st.form("exercise_form", clear_on_submit=True):
        ex_name = st.text_input("Exercise Name / Number", placeholder="e.g., Exercise 1 or Chapter 2 Set A")
        ex_range = st.text_input("Slide/Page Range", placeholder="e.g., Slide 1 to 14 or Pages 20-35")
        ex_desc = st.text_area("Key Formulae / Focus", placeholder="Notes on problem types covered in this range...")
        submit_ex = st.form_submit_button("Add Exercise Mapping")

        if submit_ex and ex_name:
            topic_data["exercises"].append({"name": ex_name, "range": ex_range, "desc": ex_desc})
            save_data(st.session_state.db)
            st.success("Exercise mapping added!")
            st.rerun()

    if topic_data["exercises"]:
        st.write("### Saved Exercise Ranges")
        for i, ex in enumerate(topic_data["exercises"]):
            with st.expander(f"📌 {ex['name']} — Range: {ex['range']}"):
                st.write(f"**Details / Focus:** {ex['desc']}")
                if st.button(f"Delete {ex['name']}", key=f"del_ex_{i}"):
                    topic_data["exercises"].pop(i)
                    save_data(st.session_state.db)
                    st.rerun()

# 4. Written Notes / Theory
with tab_notes:
    st.subheader("Extra Theory & Topic Notes")
    notes_input = st.text_area(
        "Write markdown notes, quick formulas, or short summaries:", 
        value=topic_data.get("notes_text", ""), 
        height=250
    )
    if st.button("Save Written Notes"):
        topic_data["notes_text"] = notes_input
        save_data(st.session_state.db)
        st.success("Notes saved successfully!")

# 5. Q&A Practice Sets
with tab_qa:
    st.subheader("Topic Question-Answer Practice Set")
    with st.form("qa_form", clear_on_submit=True):
        question = st.text_area("Question / Problem Prompt:")
        answer = st.text_area("Answer / Detailed Solution:")
        submit_qa = st.form_submit_button("Add Q&A Pair")
        
        if submit_qa and question and answer:
            topic_data["qa_list"].append({"q": question, "a": answer})
            save_data(st.session_state.db)
            st.success("Q&A pair saved!")
            st.rerun()

    if topic_data["qa_list"]:
        st.write("### Saved Questions & Solutions")
        for i, qa in enumerate(topic_data["qa_list"]):
            with st.expander(f"Q{i+1}: {qa['q'][:80]}..."):
                st.markdown(f"**Question:**\n{qa['q']}")
                st.markdown("---")
                st.markdown(f"**Solution / Explanation:**\n{qa['a']}")
                if st.button(f"Delete Q{i+1}", key=f"del_qa_{i}"):
                    topic_data["qa_list"].pop(i)
                    save_data(st.session_state.db)
                    st.rerun()
