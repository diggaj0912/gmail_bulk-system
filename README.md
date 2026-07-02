# gmail_bulk-system

# CODE-1

from fpdf import FPDF
import pandas as pd
import os
import smtplib
from email.mime.multipart import MIMEMultipart
from email.mime.text import MIMEText
from email.mime.base import MIMEBase
from email import encoders

# ===================== CERTIFICATE GENERATOR ===================== #
class CertificateGenerator(FPDF):
    def _init_(self):
        super()._init_()
        # Try to load OpenSans, else fallback to Arial
        try:
            self.add_font('OpenSans', '', 'OpenSans-Regular.ttf', uni=True)
            self.add_font('OpenSans', 'B', 'OpenSans-Bold.ttf', uni=True)
            self.default_font = "OpenSans"
        except:
            self.default_font = "Arial"
    
    def header(self):
        """Background image for every page"""
        try:
            self.image("certificate_template.png", x=0, y=0, w=210, h=297)  # A4
        except:
            print("⚠  certificate_template.png not found, using blank background")

    def add_text(self, name, designation, sr, program_semester_data, semester="May-2024"):
        """Add personalized certificate content"""
        self.set_font(self.default_font, "", 13)

        # Certificate No. (top left)
        self.set_xy(20, 50)
        self.cell(0, 10, f"COE/GUG/2025/209 (1-680)/{sr}", ln=1)

        # Main Body
        self.set_xy(17, 68)
        self.multi_cell(
            0, 9,
            f"This is to certify that {name}, {designation}, has set the Question Paper "
            f"for the below-mentioned Program(s) during {semester} examinations. "
            f"The Question Paper was meticulously prepared, keeping in mind the "
            f"curriculum requirements and University instructions.",
            align='L'
        )

        # Parse program-semester data
        all_programs, all_semesters = [], []
        if pd.notna(program_semester_data) and str(program_semester_data).strip():
            programs_list = [p.strip() for p in str(program_semester_data).split(';') if p.strip()]
            for program_item in programs_list:
                if '(' in program_item and ')' in program_item:
                    last_open = program_item.rfind('(')
                    program_name = program_item[:last_open].strip()
                    semester_num = program_item[last_open+1:program_item.rfind(')')].strip()
                else:
                    program_name, semester_num = program_item, "1"
                all_programs.append(program_name)
                all_semesters.append(semester_num)

        # Create program table
        self.create_program_table_with_semesters(all_programs, all_semesters)

    def create_program_table_with_semesters(self, programs, semesters, start_x=15, start_y=115):
        """Draws a clean program table with semesters"""
        self.set_xy(start_x, start_y)
        self.set_font(self.default_font, "B", 12)

        # Table column widths
        col1_width, col2_width, col3_width = 25, 110, 50
        row_height = 8

        # Header row
        self.cell(col1_width, row_height, "Sr.No.", border=1, align='C')
        self.cell(col2_width, row_height, "Program Name", border=1, align='C')
        self.cell(col3_width, row_height, "Semester(s)", border=1, align='C', ln=1)

        # Body rows
        self.set_font(self.default_font, "", 11)
        for i, (program, semester) in enumerate(zip(programs, semesters), 1):
            self.set_x(start_x)
            self.cell(col1_width, row_height, str(i), border=1, align='C')
            self.cell(col2_width, row_height, program, border=1, align='L')
            self.cell(col3_width, row_height, semester, border=1, align='C', ln=1)

        # Add signature below table
        table_end_y = start_y + (len(programs) + 1) * row_height
        signature_x, signature_y = start_x + 70, table_end_y + 15

        try:
            self.image("Signature.png", x=signature_x, y=signature_y, w=50, h=25)
        except:
            print("⚠  Signature.png not found, skipping signature")

        self.set_font(self.default_font, "B", 12)
        self.set_xy(signature_x, signature_y + 28)
        self.cell(50, 10, "Controller of Examinations", align='C')

# ===================== EMAIL FUNCTION ===================== #
def send_certificate_email(recipient_email, recipient_name, certificate_file):
    """Send certificate via email"""
    sender_email = "techlearner248@gmail.com"   # your email
    sender_password = os.getenv("EMAIL_APP_PASSWORD")  # app password (stored in env)

    if not sender_password:
        print("❌ No EMAIL_APP_PASSWORD found in environment. Skipping email.")
        return False
    
    # Create message
    msg = MIMEMultipart()
    msg['From'] = sender_email
    msg['To'] = recipient_email
    msg['Subject'] = "Certificate of Question Paper Setting"
    
    body = f"""
    Dear {recipient_name},

    Please find attached your certificate for setting the Question Paper during May-2025 examinations.

    This certificate acknowledges your contribution to the academic excellence of our institution.

    Best regards,  
    Controller of Examinations
    """
    msg.attach(MIMEText(body, 'plain'))
    
    # Attach file
    with open(certificate_file, "rb") as attachment:
        part = MIMEBase('application', 'octet-stream')
        part.set_payload(attachment.read())
    encoders.encode_base64(part)
    part.add_header(
        'Content-Disposition',
        f'attachment; filename={os.path.basename(certificate_file)}'
    )
    msg.attach(part)
    
    # Send email
    try:
        server = smtplib.SMTP('smtp.gmail.com', 587)
        server.starttls()
        server.login(sender_email, sender_password)
        server.sendmail(sender_email, recipient_email, msg.as_string())
        server.quit()
        print(f"✅ Certificate sent successfully to {recipient_email}")
        return True
    except Exception as e:
        print(f"❌ Failed to send email to {recipient_email}: {str(e)}")
        return False

# ===================== MAIN SCRIPT ===================== #
if _name_ == "_main_":
    data = pd.read_csv("output.csv")

    for idx, row in data.iterrows():
        pdf = CertificateGenerator()
        pdf.add_page()

        sr_number = idx + 1  # unique certificate number

        pdf.add_text(
            row['Name'],
            row['Designation'],
            sr_number,
            row['Program & Semester'],
            row.get('Semester', 'May-2024')
        )

        # Save certificate with clean file name
        person_name = "".join(c if c.isalnum() or c == "" else "" for c in row['Name'])
        filename = f"certificate_{person_name}.pdf"
        pdf.output(filename)

        # Send email if valid
        recipient_email = row.get('Email', '')
        if recipient_email and recipient_email.lower() != 'unknown':
            print(f"📧 Sending certificate to {row['Name']} at {recipient_email}...")
            send_certificate_email(recipient_email, row['Name'], filename)
        else:
            print(f"⚠ No valid email for {row['Name']}, saved as {filename}")

    print("✅ All certificates generated successfully!")

    
