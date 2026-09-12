import os
import requests
import time  
import numpy as np
import sounddevice as sd
import speech_recognition as sr
import pyttsx3  
import msvcrt  
from scipy.io.wavfile import read, write  
from transformers import pipeline  

# ============================================================
# 오디오 입출력 장치 설정
# ============================================================

# 개발 당시 사용한 마이크 / 스피커 장치를 직접 지정한 값입니다.
# PC마다 장치 번호가 다를 수 있으므로 실행 환경에 맞게 수정해야 합니다.

sd.default.device = (1, 5)  
SAMPLE_RATE = 16000  
CHANNELS = 1         

# TTS 음성을 출력할 노트북 스피커 장치 번호
LAPTOP_SPEAKER_ID = 5 

# ============================================================
# Telegram 설정
# ============================================================

# 실제 실행 시 본인의 Telegram Bot Token과 Chat ID를 입력
# 공개 저장소에는 보안상 실제 인증 정보를 포함하지 않음

BOT_TOKEN = "..."
CHAT_ID = "..."

AUTO_ON_TIMEOUT = 5  
VOLUME_THRESHOLD = 0.1

CRITICAL_KEYWORDS = ["살려", "119", "쓰러", "숨못", "죽겠", "죽네", "죽는다", "아파", "아프다", "못일어"]
SAFE_KEYWORDS = ["덥다", "더워", "춥다", "추워", "짜증", "귀찮", "심심", "시원", "하하", "히히", "헤헤"]


def send_telegram_alert(message):
    url = f"https://api.telegram.org/bot{BOT_TOKEN}/sendMessage"
    payload = {"chat_id": CHAT_ID, "text": message}
    try:
        response = requests.post(url, json=payload)
        return response.status_code == 200
    except Exception as e:
        print(f"🚨 [통신 에러] 텔레그램 전송 실패: {e}")
    return False


def speak_guidance(text):
    wav_filename = "tts_temp.wav"
    try:
        engine = pyttsx3.init()
        engine.setProperty('rate', 160) 
        engine.save_to_file(text, wav_filename)
        engine.runAndWait() 
        
        fs, data = read(wav_filename)
        
        if data.ndim == 1:
            data = np.column_stack((data, data))
            
        sd.play(data, fs, device=LAPTOP_SPEAKER_ID)
        sd.wait()  
    except Exception as e:
        print(f"🚨 [TTS 라우팅 에러] 노트북 스피커 음성 출력 실패: {e}")
    finally:
        if os.path.exists(wav_filename):
            os.remove(wav_filename)


def listen_for_loud_sound_safe(threshold=VOLUME_THRESHOLD, sample_rate=SAMPLE_RATE):
    chunk_duration = 0.2  
    print("\n🟢 [ON] 실시간 독거노인 소음 모니터링 가동 중...")
    print(f"💡 (가족 방문 시 'p' = 일시 정지 | 시스템 완전 안전 종료 시 'q' 를 누르세요!)")
    
    while True:
        if msvcrt.kbhit():
            key = msvcrt.getch().decode('utf-8', errors='ignore').lower()
            
            if key == 'p':
                print("\n⏸️ [OFF - 가족 방문 모드] 시스템이 일시 정지되었습니다. (알림 유예)")
                pause_start_time = time.time()
                
                while True:
                    if msvcrt.kbhit():
                        resume_key = msvcrt.getch().decode('utf-8', errors='ignore').lower()
                        if resume_key == 'r':
                            print("\n🟢 [ON - 수동 복귀] 사용자가 직접 시스템을 다시 재가동했습니다.\n")
                            break
        
                        elif resume_key == 'q':
                            print("\n🛑 [종료 인터럽트] 일시정지 상태에서 시스템 종료 명령이 접수되었습니다.")
                            speak_guidance("안심 케어 시스템을 정상 종료합니다.")
                            print("👋 [SYSTEM SHUTDOWN] 프로그램을 안전하게 닫습니다.\n")
                            os._exit(0)
                    
                    elapsed_time = time.time() - pause_start_time
                    remaining_time = int(AUTO_ON_TIMEOUT - elapsed_time)
                    
                    if remaining_time >= 0:
                        print(f"⏳ 자동 재가동 대기 중... (남은 시간: {remaining_time}초)  ", end="\r", flush=True)
                    
                    if elapsed_time >= AUTO_ON_TIMEOUT:
                        print(f"\n\n⏰ [자동 ON - 타임아웃 복구] 설정된 일시정지 시간({AUTO_ON_TIMEOUT}초)이 만료되었습니다.")
                        print("📢 어르신의 안전을 위해 시스템이 스스로 모니터링을 재개합니다.")
                        
                        speak_guidance("안심 케어 시스템 모니터링을 다시 시작합니다.")
                        print("🟢 [ON - 모니터링 재개] 시스템 가동 정상화\n")
                        break
                        
                    sd.sleep(200) 

            elif key == 'q':
                print("\n🛑 [종료 인터럽트] 사용자에 의해 안심 케어 시스템 종료 명령이 접수되었습니다.")
                speak_guidance("안심 케어 시스템을 정상 종료합니다.")
                print("👋 [SYSTEM SHUTDOWN] 프로그램을 안전하게 닫습니다.\n")
                os._exit(0)

        recording = sd.rec(int(chunk_duration * sample_rate), samplerate=sample_rate, channels=CHANNELS, dtype='float32')
        sd.wait()
        
        rms = np.sqrt(np.mean(recording**2))
        print(f"🎤 실시간 소음 수치 (RMS): {rms:.4f} / 트리거 기준치: {threshold}  ", end="\r", flush=True)

        if rms > threshold:
            print(f"\n💥 [충격음 감지 완료] 기준치 이상의 큰 소리가 발생했습니다! (감지치: {rms:.4f})")
            break


def record_and_stt(duration=4, sample_rate=SAMPLE_RATE):
    print(f"[음성 인식 시작]")
    recording = sd.rec(int(duration * sample_rate), samplerate=sample_rate, channels=CHANNELS, dtype='float32')
    sd.wait()
    print("🛑 [녹음 완료] 음성을 분석하는 중입니다...")

    audio_int16 = (recording * 32767).astype(np.int16)
    wav_filename = "temp_voice.wav"
    write(wav_filename, sample_rate, audio_int16)

    recognizer = sr.Recognizer()
    try:
        with sr.AudioFile(wav_filename) as source:
            Guided_audio = recognizer.record(source)
        text = recognizer.recognize_google(Guided_audio, language="ko-KR")
        print(f"🗣️ 어르신의 답변: \"{text}\"")
        return text
    except Exception:
        print("❌ 소리가 들리지 않거나 음성 추출에 실패했습니다.")
        return None
    finally:
        if os.path.exists(wav_filename):
            os.remove(wav_filename)


def analyze_elderly_situation(user_speech, classifier):
    clean_speech = user_speech.replace(" ", "")
    
    if any(ck in clean_speech for ck in CRITICAL_KEYWORDS):
        print("핵심 위험 키워드 즉시 매칭! (AI 분석 생략)")
        msg = f"🚨 [자동 감지 구조 요청]\n독거노인 거주지에서 충격음 발생 후 위급 발화 포착!\n\n▶ 인식된 발화: '{user_speech}'\n▶ 판정 근거: 위험 키워드 즉시 매칭"
        send_telegram_alert(msg)
        return

    if any(sk in clean_speech for sk in SAFE_KEYWORDS):
        print("✅ [가드레일 단계] 일상적인 투정 및 안전 문장으로 판단하여 상황을 종료합니다.")
        return

    print("🤔 가드레일 통과. AI가 문장의 부정 문맥 여부를 정밀 분석 중입니다...")
    try:
        result = classifier(user_speech)[0]
        label = result['label']
        score = result['score']
        
        print(f"🤖 [AI 분석 결과] 문맥 판정: {label} (신뢰도: {score*100:.1f}%)")
        
        if label == "LABEL_0" and score >= 0.85:
            print("⚠️ [AI 위급 판정] 위험 문맥 확정! 텔레그램 알림을 발송합니다.")
            msg = f"🚨 [AI 자동 경보]\n충격음 감지 후 어르신의 답변에서 위험 문맥 포착!\n\n▶ 인식된 발화: '{user_speech}'\n▶ AI 확신도: {score*100:.1f}%\n▶ 판정 근거: BERT 모델 맥락 추론"
            send_telegram_alert(msg)
        else:
            print(f"✅ [안전] 소음은 발생했으나 어르신의 음성 분석 결과 정상 상태로 판정되었습니다.")
    except Exception as e:
        print(f"🚨 AI 추론 중 오류 발생: {e}")


if __name__ == "__main__":
    print("=" * 60)
    print("⏳ 시스템 안정형 BERT AI 엔진 로드 중...")
    ai_engine = pipeline("sentiment-analysis", model="matthewburke/korean_sentiment")
    print("✅ [시스템 구동 완료] 실시간 마이크 감지 기반 지능형 돌봄 시스템 가동!")
    print("=" * 60)
    
    while True:
        listen_for_loud_sound_safe(threshold=VOLUME_THRESHOLD)
        
        print("\n📢 [스피커 안내 멘트 발송] ===========================")
        print("🤖 AI 시스템: \"어르신! 괜찮으세요?\"")
        print("====================================================\n")
        
        speak_guidance("어르신 괜찮으세요?")
        
        speech_text = record_and_stt(duration=4)
        
        if speech_text:
            analyze_elderly_situation(speech_text, ai_engine)
        else:
            print("ℹ️ 어르신의 답변이 인식되지 않아 상황을 유예하고 다음 모니터링을 준비합니다.")
        
        print("\n🔄 3초 후 다시 실시간 거주지 소음 모니터링 모드로 복귀합니다...")
        sd.sleep(3000)
