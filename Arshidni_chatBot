import mysql.connector
import spacy
from google import genai
from google.genai.errors import APIError

# 1. الإعدادات الأولية والاتصال بقاعدة البيانات

# قيم تحتاج التحديث
DB_CONFIG = {
    'host': '127.0.0.1',
    'user': 'root',
    'password': 'xxxxx',
    'database': 'Abshir_Guide_DB'
}

# إعدادات LLM (Gemini API)
GEMINI_API_KEY = "xxxxx"  
MODEL_NAME_GEMINI = "gemini-2.5-flash"  

def get_db_connection():
    """تنشئ اتصالاً جديداً بقاعدة البيانات."""
    try:
        return mysql.connector.connect(**DB_CONFIG)
    except mysql.connector.Error as err:
        print(f"خطأ في الاتصال بقاعدة البيانات: {err}")
        return None

# تحميل نموذج اللغة (لتحليل بسيط للنص)
try:
    nlp = spacy.load("en_core_web_sm") 
except OSError:
    print("SpaCy نموذج اللغة غير محمل. سيتم الاعتماد على البحث المباشر بالكلمات المفتاحية.")
    nlp = None

# 2. وظائف التحليل والبحث عن الخدمة/الرحلة (بدون تغيير)

def analyze_user_input(user_input):
    cleaned_input = user_input.strip() 
    if nlp:
        doc = nlp(cleaned_input)
        keywords = [token.text for token in doc if token.pos_ in ('NOUN', 'PROPN', 'ADJ')]
        if keywords:
            return " ".join(keywords) 
    return cleaned_input

def find_target_item(search_term):
    conn = get_db_connection()
    if not conn:
        return None, None
        
    cursor = conn.cursor(dictionary=True)
    target_id, id_type, results = None, None, None
    
    try:
        search_journey_query = """
        SELECT journey_id FROM Complex_Journeys
        WHERE search_keyword = %s OR journey_name LIKE %s
        """
        cursor.execute(search_journey_query, (search_term, f'%{search_term}%'))
        journey = cursor.fetchone()
        
        # الإصلاح لخطأ MySQL 1
        if cursor.description is not None:
             cursor.fetchall() 

        if journey:
            target_id = journey['journey_id']
            id_type = 'journey_id'
        else:
            search_service_query = """
            SELECT service_id FROM Services
            WHERE service_name = %s OR service_name LIKE %s
            """
            cursor.execute(search_service_query, (search_term, f'%{search_term}%'))
            service = cursor.fetchone()
            
            # الإصلاح لخطأ MySQL 2
            if cursor.description is not None:
                 cursor.fetchall() 
            
            if service:
                target_id = service['service_id']
                id_type = 'service_id'

        if not target_id:
            return None, None
            
        main_query = """
        SELECT
            J.journey_name, S.service_name, E.entity_name, E.entity_url,
            ST.step_order, ST.step_description,
            RD.req_display_name_ar, RD.source_type, SR.is_required, S.service_id
        FROM
            Services S
        JOIN Entities E ON S.entity_id = E.entity_id
        LEFT JOIN Journey_Services JS ON S.service_id = JS.service_id
        LEFT JOIN Complex_Journeys J ON JS.journey_id = J.journey_id
        LEFT JOIN Steps ST ON S.service_id = ST.service_id
        LEFT JOIN Service_Requirements SR ON ST.step_id = SR.step_id
        LEFT JOIN Requirement_Definition RD ON SR.req_def_id = RD.req_def_id
        """
        
        if id_type == 'service_id':
            where_clause = "WHERE S.service_id = %s"
        else:
            where_clause = "WHERE J.journey_id = %s"

        order_clause = """
        ORDER BY
            J.journey_id, JS.service_order, ST.step_order;
        """

        final_query = main_query + where_clause + order_clause
        
        cursor.execute(final_query, (target_id,))
        results = cursor.fetchall()

        return results, id_type

    except Exception as e:
        print(f"خطأ أثناء تنفيذ الاستعلام: {e}")
        return None, None
    finally:
        cursor.close()
        conn.close()

# 3. دالة بناء السياق (RAG Context Builder)

def build_rag_context(results, id_type):
    if not results:
        return "لا تتوفر بيانات محددة لهذه الخدمة/الرحلة."
    # (الكود المتبقي لبناء السياق لم يتغير)
    context_data = []
    title = results[0]['journey_name'] if id_type == 'journey_id' else results[0]['service_name']
    context_data.append(f"## البيانات المستخرجة من قاعدة بيانات الدليل الحكومي لـ: {title}")
    current_service = None
    for row in results:
        if row['service_name'] != current_service:
            current_service = row['service_name']
            context_data.append(f"\n--- [خدمة: {current_service}] ---")
            context_data.append(f"الجهة: {row['entity_name']}. الرابط: {row['entity_url']}")
        step_text = f"الخطوة {row['step_order']}: {row['step_description']}"
        req_list = []
        for req_row in results:
            if req_row['step_description'] == row['step_description'] and req_row['service_name'] == current_service:
                if req_row['req_display_name_ar']:
                    status = "إلزامي" if req_row['is_required'] else "اختياري"
                    req_list.append(f"{req_row['req_display_name_ar']} ({status} - المصدر: {req_row['source_type']})")

        context_data.append(f"\n{step_text}")
        if req_list:
            context_data.append("المتطلبات التفصيلية: " + " | ".join(req_list))
        else:
            context_data.append("المتطلبات التفصيلية: لا يوجد.")

    return "\n".join(context_data)

# دالة جديدة: تصنيف نية المستخدم (Intent Classification)

def classify_user_intent(user_query):
    """
    تستخدم Gemini لتحديد ما إذا كانت نية المستخدم هي محادثة عامة أو استفسار عن خدمة.
    النتيجة المتوقعة: 'SERVICE_QUERY' أو 'GENERAL_CHAT'.
    """
    try:
        client = genai.Client(api_key=GEMINI_API_KEY)
        
        classification_prompt = f"""
        اقرأ الجملة التالية. حدد هدف المستخدم الرئيسي بدقة:
        - إذا كانت الجملة سؤالاً عن خدمة حكومية، خطوة، متطلب، أو إجراء (مثل 'تجديد رخصة'، 'كيف أبني منزل؟')، أجب بـ 'SERVICE_QUERY' فقط.
        - إذا كانت الجملة ترحيباً، شكراً، أو عبارة عامة لا علاقة لها بالخدمات (مثل 'أهلاً'، 'شكراً لك'، 'كيف حالك؟')، أجب بـ 'GENERAL_CHAT' فقط.
        
        جملة المستخدم: "{user_query}"
        النية: 
        """
        
        response = client.models.generate_content(
            model=MODEL_NAME_GEMINI,
            contents=classification_prompt
        )
        return response.text.strip().upper()

    except Exception as e:
        print(f"فشل تصنيف النية: {e}")
        return 'SERVICE_QUERY' 


# 4. دالة الاتصال بنموذج LLM (الآن مركزية لاتخاذ القرار)

def query_llm_for_response(user_query, context_block, response_type):
    """
    يرسل السياق والسؤال إلى Gemini API، ويستخدم Prompt مختلف حسب نوع الرد.
    """
    try:
        client = genai.Client(api_key=GEMINI_API_KEY)
        
        # 3. بناء موجه (Prompt) النموذج بناءً على نوع الرد
        if response_type == "RAG":
            # Prompt لإنشاء رد منظم ومفصل بناءً على البيانات المستخرجة
            prompt = f"""
            أنت روبوت "أرشدني" المتخصص في الخدمات الحكومية السعودية. مهمتك هي قراءة السياق التالي واستخدامه حصراً لتوليد رد للمستخدم.
            
            التنسيق الإلزامي:
            1.  يجب أن يكون الرد في شكل **قوائم رقمية** منظمة حسب **الجهة الحكومية** (entity_name).
            2.  تحت كل جهة، يجب أن تدرج **نقاط أبجدية (أ/ب/ج)** تمثل الخطوات الرئيسية المتعلقة بالجهة.
            3.  يجب أن تضع **رابط الجهة** (entity_url) في نهاية كل قائمة جهة.
            4.  في النهاية، أضف سؤالاً ختامياً للمستخدم للتفاعل.
            5.  إذا لاحظت في سؤال المستخدم كلمات غير مرتبطة بالخدمة (مثل 'عندي استفسار بخصوص')، تجاهلها وركز على الكلمات المفتاحية للخدمة ('بناء منزل').
            
            السؤال الأصلي للمستخدم: "{user_query}"
            السياق المستخرج من قاعدة البيانات:
            ---
            {context_block}
            ---
            **ابدأ الرد مباشرة بالتنسيق المطلوب.**
            """
        
        else: # response_type == "GENERAL" (لم يتم العثور على بيانات)
            # Prompt للردود العامة، التحيات، أو الاعتذار لعدم العثور على الخدمة
            prompt = f"""
            أنت روبوت "أرشدني" الودود. لقد فشل نظام البحث في قاعدة البيانات الحكومية في العثور على خدمة تطابق "{user_query}".
            
            إذا كان سؤال المستخدم تحية أو شكر ('أهلاً'، 'شكراً')، فقم بالرد بلطف.
            إذا كان سؤال المستخدم استفساراً عن خدمة ('بناء منزل') ولم تجدها في السياق، اعتذر بلطف وذكاء، واطلب منه محاولة كلمة مفتاحية أخرى.
            
            السياق (ملاحظة: لا يحتوي على بيانات خدمة):
            ---
            {context_block}
            ---
            الرد المطلوب:
            """
        
        # 4. إرسال الطلب إلى Gemini
        response = client.models.generate_content(
            model=MODEL_NAME_GEMINI,
            contents=prompt
        )
        
        return response.text.strip()

    except APIError as e:
        return f"❌ خطأ في API Gemini: {e}"
    except Exception as e:
        return f"❌ خطأ عام: {e}"


# 5. الدالة الرئيسية لمنطق التشات بوت (RAG) (المحدثة للسرعة والذكاء)

def chat_bot_rag(user_query, session_id="rag_session"):
    """
    الدالة الرئيسية التي تنفذ عملية RAG: تبحث أولاً، ثم تقرر نوع الرد (RAG أو عام).
    """
    # 1. البحث عن الخدمة/الرحلة
    search_term = analyze_user_input(user_query)
    results, id_type = find_target_item(search_term)

    # 2. بناء السياق (سواء وجد أو لم يجد)
    if results:
        # إذا وجدت بيانات، نبني سياق الـ RAG التفصيلي
        context_block = build_rag_context(results, id_type)
        response_type = "RAG"
    else:
        # إذا لم نجد بيانات، نستخدم سياقاً فارغاً ونطلب من Gemini الرد بشكل عام
        context_block = "لا تتوفر أي بيانات محددة لهذه الكلمات المفتاحية في قاعدة البيانات الحكومية."
        response_type = "GENERAL"

    # 3. إرسال الطلب إلى LLM (استدعاء API واحد فقط)
    final_response = query_llm_for_response(user_query, context_block, response_type)
    
    return final_response

# 6. محاكاة واجهة التشغيل (للتجربة في الكونسول)

def run_console_interface():
    """
    واجهة بسيطة لتجربة التشات بوت في الكونسول.
    """
    print("🤖 أرشدني - دليل الخدمات الحكومية التفاعلي (Gemini RAG)")
    
    session_id = "rag_user"

    while True:
        user_input = input("\n👤 أنت: ")
        if user_input.lower() in ['خروج', 'إنهاء', 'exit']:
            print("شكراً لاستخدامك النظام. مع السلامة.")
            break
            
        response = chat_bot_rag(user_input, session_id)
        print(f"\n🤖 أرشدني:\n{response}")

# تشغيل الواجهة التجريبية
if __name__ == "__main__":
    run_console_interface()
