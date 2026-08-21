

import asyncio
import json
import re
import os
import sys
import time
import hashlib
import logging
import sqlite3
import threading
import queue
import random
import string
from datetime import datetime, timedelta
from dataclasses import dataclass, field
from typing import Dict, List, Optional, Tuple, Any, Set
from enum import Enum
from collections import defaultdict, Counter
from pathlib import Path

# =================================================================================
# 3rd Party Imports
# =================================================================================
try:
    from telethon import TelegramClient, errors, events
    from telethon.tl.functions.channels import (
        GetFullChannelRequest,
        GetParticipantRequest,
        GetAdminedPublicChannelsRequest
    )
    from telethon.tl.functions.messages import (
        GetHistoryRequest, 
        GetMessagesViewsRequest,
        ImportChatInviteRequest,
        ReportRequest
    )
    from telethon.tl.functions.account import UpdateProfileRequest
    from telethon.tl.types import (
        MessageMediaPhoto,
        MessageMediaDocument,
        MessageMediaWebPage,
        Channel,
        Chat,
        User,
        Message,
        InputReportReasonSpam,
        InputReportReasonViolence,
        InputReportReasonPornography,
        InputReportReasonOther
    )
    from telethon.tl.custom import Message as CustomMessage
    from telethon.utils import get_display_name
except ImportError as e:
    print(f"❌ Error importing telethon: {e}")
    print("📦 Please install: pip install telethon")
    sys.exit(1)

# =================================================================================
# Logging Configuration
# =================================================================================
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(levelname)s - %(message)s',
    handlers=[
        logging.FileHandler('telegram_ai_pro.log', encoding='utf-8'),
        logging.StreamHandler()
    ]
)
logger = logging.getLogger(__name__)

# =================================================================================
# Constants & Configuration - به‌روز شده
# =================================================================================
CONFIG = {
    'API_ID': 39130364,
    'API_HASH': 'be6d96339f96d22d846ae6e96dc12bc3',
    'MAX_MESSAGES': 500,
    'BATCH_SIZE': 50,
    'SCAN_INTERVAL': 180,  # 3 minutes (سریع‌تر)
    'MAX_RETRIES': 5,
    'TIMEOUT': 30,
    'DATABASE': 'violations_pro.db',
    'REPORT_DIR': 'reports_pro',
    'LOG_FILE': 'telegram_ai_pro.log',
    'SESSION_FILE': 'ai_session_pro'
}

# =================================================================================
# TARGET CHANNEL - کانال هدف جدید
# =================================================================================
TARGET_CHANNELS = {
    'primary': {
        'link': 'https://t.me/+fClIA2SAPwk0MDBk',
        'hash': 'fClIA2SAPwk0MDBk',
        'name': 'کانال هدف اصلی',
        'priority': 'CRITICAL'
    }
}

# =================================================================================
# Enums & Data Classes
# =================================================================================
class ViolationSeverity(Enum):
    """Severity levels for violations"""
    LOW = (1, '🟢', 'Low')
    MEDIUM = (2, '🟡', 'Medium')
    HIGH = (3, '🟠', 'High')
    CRITICAL = (4, '🔴', 'Critical')
    EXTREME = (5, '💀', 'Extreme')
    
    def __init__(self, level: int, emoji: str, label: str):
        self.level = level
        self.emoji = emoji
        self.label = label
    
    @classmethod
    def from_level(cls, level: int) -> 'ViolationSeverity':
        for severity in cls:
            if severity.level == level:
                return severity
        return cls.LOW

class ViolationType(Enum):
    """Types of violations"""
    HATE_SPEECH = ('hate_speech', 'توهین نژادی و مذهبی')
    DOXXING = ('doxxing', 'افشای اطلاعات شخصی')
    THREAT = ('threat', 'تهدید به خشونت')
    SPAM = ('spam', 'اسپم و تبلیغات')
    EXPLICIT = ('explicit', 'محتوای بزرگسالان')
    HARASSMENT = ('harassment', 'آزار و اذیت')
    IMPERSONATION = ('impersonation', 'جعل هویت')
    SCAM = ('scam', 'کلاهبرداری')
    PHISHING = ('phishing', 'فیشینگ')
    MALWARE = ('malware', 'بدافزار')
    CHILD_ABUSE = ('child_abuse', 'آزار کودکان')
    TERRORISM = ('terrorism', 'تروریسم')
    DRUGS = ('drugs', 'مواد مخدر')
    GAMBLING = ('gambling', 'قمار')
    ILLEGAL = ('illegal', 'فعالیت غیرقانونی')
    TARGET_SPECIFIC = ('target_specific', 'تخلف ویژه کانال هدف')
    
    def __init__(self, code: str, label: str):
        self.code = code
        self.label = label

@dataclass
class Violation:
    """Data class for a single violation"""
    type: ViolationType
    severity: ViolationSeverity
    keyword: str
    context: str
    message_id: int
    sender_id: int
    sender_username: Optional[str]
    timestamp: datetime
    confidence: float  # 0.0 to 1.0
    
    def to_dict(self) -> Dict:
        return {
            'type': self.type.code,
            'type_label': self.type.label,
            'severity': self.severity.label,
            'severity_level': self.severity.level,
            'keyword': self.keyword,
            'context': self.context,
            'message_id': self.message_id,
            'sender_id': self.sender_id,
            'sender_username': self.sender_username,
            'timestamp': self.timestamp.isoformat(),
            'confidence': self.confidence
        }

@dataclass
class ChannelInfo:
    """Data class for channel information"""
    id: int
    username: str
    title: str
    link: str
    members_count: int
    about: str
    is_private: bool
    is_verified: bool
    is_scam: bool
    is_fake: bool
    slow_mode: bool
    created_date: Optional[datetime]
    violations: List[Violation] = field(default_factory=list)
    scan_timestamp: datetime = field(default_factory=datetime.now)
    is_target: bool = False
    
    def to_dict(self) -> Dict:
        return {
            'id': self.id,
            'username': self.username,
            'title': self.title,
            'link': self.link,
            'members_count': self.members_count,
            'about': self.about,
            'is_private': self.is_private,
            'is_verified': self.is_verified,
            'is_scam': self.is_scam,
            'is_fake': self.is_fake,
            'slow_mode': self.slow_mode,
            'created_date': self.created_date.isoformat() if self.created_date else None,
            'violations': [v.to_dict() for v in self.violations],
            'scan_timestamp': self.scan_timestamp.isoformat(),
            'is_target': self.is_target
        }

# =================================================================================
# Target Channel Scanner - اسکنر ویژه کانال هدف
# =================================================================================
class TargetChannelScanner:
    """اسکنر اختصاصی برای کانال‌های هدف با لینک دعوت"""
    
    def __init__(self, client):
        self.client = client
        self.target_link = TARGET_CHANNELS['primary']['link']
        self.target_hash = TARGET_CHANNELS['primary']['hash']
        self.found_channel = None
        
    async def join_and_scan(self, max_messages: int = 500) -> Optional[ChannelInfo]:
        """ورود به کانال با لینک دعوت و اسکن کامل"""
        try:
            logger.info(f"🔍 در حال ورود به کانال هدف: {self.target_link}")
            
            # ورود به کانال
            result = await self.client(ImportChatInviteRequest(self.target_hash))
            
            # پیدا کردن کانال
            if hasattr(result, 'chat'):
                channel = result.chat
                logger.info(f"✅ با موفقیت وارد کانال شدید: {channel.title}")
                self.found_channel = channel
                return await self._scan_channel(channel, max_messages)
            else:
                logger.error("❌ خطا در ورود به کانال")
                return None
                
        except errors.rpcerrorlist.InviteHashExpiredError:
            logger.error("❌ لینک دعوت منقضی شده است!")
            return None
        except errors.rpcerrorlist.InviteHashInvalidError:
            logger.error("❌ لینک دعوت نامعتبر است!")
            return None
        except errors.rpcerrorlist.UserAlreadyParticipantError:
            logger.info("⚠️ شما قبلاً در این کانال عضو هستید!")
            # پیدا کردن از دیالوگ‌ها
            return await self._find_from_dialogs(max_messages)
        except Exception as e:
            logger.error(f"❌ خطا: {e}")
            return None
            
    async def _find_from_dialogs(self, max_messages: int) -> Optional[ChannelInfo]:
        """پیدا کردن کانال از دیالوگ‌ها"""
        try:
            async for dialog in self.client.iter_dialogs():
                if dialog.is_channel:
                    # جستجو برای کانال هدف
                    if "چایخانه" in dialog.name or "ابولولو" in dialog.name:
                        logger.info(f"✅ کانال پیدا شد: {dialog.name}")
                        self.found_channel = dialog.entity
                        return await self._scan_channel(dialog.entity, max_messages)
            logger.error("❌ کانال در دیالوگ‌ها پیدا نشد!")
            return None
        except Exception as e:
            logger.error(f"❌ خطا در جستجوی دیالوگ‌ها: {e}")
            return None
            
    async def _scan_channel(self, channel, max_messages: int) -> ChannelInfo:
        """اسکن کانال پیدا شده"""
        try:
            # دریافت اطلاعات کامل
            full_info = await self.client(GetFullChannelRequest(channel=channel))
            
            channel_info = ChannelInfo(
                id=channel.id,
                username=channel.username if hasattr(channel, 'username') else 'target_channel',
                title=channel.title,
                link=f"https://t.me/{channel.username}" if hasattr(channel, 'username') else self.target_link,
                members_count=full_info.full_chat.participants_count,
                about=full_info.full_chat.about or "بدون توضیحات",
                is_private=True,
                is_verified=False,
                is_scam=False,
                is_fake=False,
                slow_mode=hasattr(full_info.full_chat, 'slowmode_seconds') and full_info.full_chat.slowmode_seconds > 0,
                created_date=channel.date if hasattr(channel, 'date') else None,
                is_target=True
            )
            
            logger.info(f"📊 کانال: {channel.title} ({full_info.full_chat.participants_count} عضو)")
            
            # دریافت پیام‌ها
            logger.info(f"📨 دریافت {max_messages} پیام آخر...")
            messages = await self.client(GetHistoryRequest(
                peer=channel,
                limit=max_messages,
                offset_id=0,
                offset_date=None,
                add_offset=0,
                max_id=0,
                min_id=0,
                hash=0
            ))
            
            # تحلیل پیام‌ها
            logger.info("🔎 تحلیل پیام‌ها...")
            detector = ViolationDetector()
            all_violations = []
            
            for msg in messages.messages:
                if not msg.message:
                    continue
                    
                sender_username = None
                if msg.sender_id:
                    try:
                        sender = await self.client.get_entity(msg.sender_id)
                        sender_username = sender.username
                    except:
                        pass
                        
                violations = detector.analyze_message(
                    msg.message,
                    msg.id,
                    msg.sender_id or 0,
                    sender_username,
                    msg.date
                )
                
                all_violations.extend(violations)
                
            channel_info.violations = all_violations
            channel_info.scan_timestamp = datetime.now()
            
            logger.info(f"✅ اسکن کامل شد: {len(all_violations)} تخلف یافت شد")
            
            return channel_info
            
        except Exception as e:
            logger.error(f"❌ خطا در اسکن کانال: {e}")
            return None
            
    async def auto_report(self, channel_info: ChannelInfo):
        """ریپورت خودکار تخلفات کانال هدف"""
        if not channel_info or not channel_info.violations:
            logger.info("✅ هیچ تخلفی برای ریپورت وجود ندارد")
            return
            
        logger.info(f"📤 ارسال ریپورت برای {len(channel_info.violations)} تخلف...")
        
        # گروه‌بندی پیام‌ها
        message_ids = list(set([v.message_id for v in channel_info.violations]))
        
        for i in range(0, len(message_ids), 20):
            batch = message_ids[i:i+20]
            try:
                await self.client(ReportRequest(
                    peer=self.found_channel,
                    id=batch,
                    reason=InputReportReasonOther(),
                    message="This channel contains hate speech, harassment, and explicit content violating Telegram's Terms of Service."
                ))
                logger.info(f"✅ ریپورت شد: {len(batch)} پیام")
                await asyncio.sleep(3)
            except Exception as e:
                logger.error(f"❌ خطا در ریپورت گروهی: {e}")
                # ریپورت تک تک
                for msg_id in batch:
                    try:
                        await self.client(ReportRequest(
                            peer=self.found_channel,
                            id=[msg_id],
                            reason=InputReportReasonOther(),
                            message="Violation of Telegram Terms of Service."
                        ))
                        logger.info(f"✅ ریپورت شد: پیام {msg_id}")
                        await asyncio.sleep(1)
                    except Exception as e2:
                        logger.error(f"❌ خطا برای پیام {msg_id}: {e2}")

# =================================================================================
# Violation Detection Engine (همان نسخه قبلی با کلمات کلیدی بیشتر)
# =================================================================================
class ViolationDetector:
    """Advanced violation detection engine with ML-like capabilities"""
    
    # Comprehensive keyword database - به‌روز شده
    KEYWORDS = {
        ViolationType.HATE_SPEECH: {
            'keywords': [
                'کصننه', 'مسلمونا', 'تهرانیا', 'جنوبیا', 'اهوازیا', 'اصفهانیا',
                'کرد', 'لر', 'بلوچ', 'عرب', 'ترک', 'فارس', 'آذری', 'گیلک',
                'مازنی', 'خراسانی', 'یزدی', 'شیرازی', 'کاشانی', 'قم', 'مشهد',
                'کص', 'کون', 'گوه', 'مادرجنده', 'حرامزاده', 'لعنتی', 'نژاد',
                'فاشیست', 'تروریست', 'شیعه', 'سنی', 'یهودی', 'مسیحی', 'زرتشتی',
                'بهائی', 'افغان', 'پاکستانی', 'عراقی', 'عربستانی', 'ترکیه',
                'کافر', 'ملحد', 'بیدین', 'بی‌دین', 'مشرک', 'بت‌پرست',
                'جنسیت', 'زن', 'مرد', 'دگرباش', 'همجنس', 'لزبین', 'گِی',
                'ترنس', 'کویر', 'ال‌جی‌بی‌تی', 'همجنس‌گرا', 'فاحشه', 'روسپی'
            ],
            'weight': 3.0,
            'severity': ViolationSeverity.HIGH
        },
        # ... بقیه KEYWORDS مثل قبل ...
    }
    
    def __init__(self):
        self.keyword_cache = {}
        self.compiled_patterns = {}
        self._compile_patterns()
        
    def _compile_patterns(self):
        for name, pattern in self.CONTEXT_PATTERNS.items():
            self.compiled_patterns[name] = re.compile(pattern, re.IGNORECASE)
            
    def analyze_message(self, message_text: str, message_id: int, 
                       sender_id: int, sender_username: Optional[str],
                       timestamp: datetime) -> List[Violation]:
        # ... (همان کد قبلی)
        pass

# =================================================================================
# Main AI Scanner PRO - نسخه ارتقا یافته
# =================================================================================
class AIScannerPro:
    """Main AI scanner for Telegram channels with target detection"""
    
    def __init__(self):
        self.client = None
        self.detector = ViolationDetector()
        self.report_generator = ReportGenerator()
        self.db_manager = DatabaseManager()
        self.scanning = False
        self.channels = {}
        self.target_scanner = None
        
    async def initialize(self):
        """Initialize the Telegram client"""
        self.client = TelegramClient(
            CONFIG['SESSION_FILE'],
            CONFIG['API_ID'],
            CONFIG['API_HASH']
        )
        await self.client.start()
        logger.info("✅ Connected to Telegram")
        self.target_scanner = TargetChannelScanner(self.client)
        
    async def scan_target_channel(self):
        """اسکن کانال هدف"""
        logger.info("🎯 اسکن کانال هدف...")
        result = await self.target_scanner.join_and_scan(CONFIG['MAX_MESSAGES'])
        
        if result:
            # ذخیره در دیتابیس
            self.db_manager.save_channel(result)
            self.db_manager.save_violations(result.id, result.violations)
            self.channels['target'] = result
            
            # نمایش گزارش
            report = self.report_generator.generate_report(result, 'standard')
            print("\n" + "="*70)
            print(report)
            print("="*70)
            
            # ذخیره گزارش
            report_path = self.report_generator.output_dir / f"target_report_{datetime.now().strftime('%Y%m%d_%H%M%S')}.txt"
            with open(report_path, 'w', encoding='utf-8') as f:
                f.write(report)
            logger.info(f"📁 Report saved: {report_path}")
            
            # ریپورت خودکار
            await self.target_scanner.auto_report(result)
            
        return result
        
    async def interactive_menu(self):
        """Interactive menu system with target option"""
        while True:
            print("\n" + "="*70)
            print("🤖   AI Telegram Channel Scanner PRO v6.0")
            print("🔥   ویژه کانال‌های خصوصی با لینک دعوت")
            print("="*70)
            print("1. 🎯 اسکن کانال هدف (https://t.me/+fClIA2SAPwk0MDBk)")
            print("2. 🔍 اسکن یک کانال (با نام کاربری)")
            print("3. 📊 اسکن چند کانال (با کاما جدا کنید)")
            print("4. 📋 مشاهده آخرین اسکن")
            print("5. 📧 تولید گزارش ایمیل")
            print("6. 📈 آمار کلی")
            print("7. 🔄 اسکن خودکار (هر ۳ دقیقه)")
            print("8. 💾 خروجی گزارش JSON")
            print("9. 🗑️ پاک کردن کش")
            print("10. ❌ خروج")
            print("="*70)
            
            choice = input("▶ انتخاب کنید: ")
            
            if choice == '1':
                await self.scan_target_channel()
                
            elif choice == '2':
                username = input("نام کاربری کانال (بدون @): ")
                await self.scan_channel(username)
                
            elif choice == '3':
                input_str = input("نام کانال‌ها (با کاما جدا کنید): ")
                usernames = [u.strip() for u in input_str.split(',')]
                await self.scan_multiple(usernames)
                
            elif choice == '4':
                username = input("نام کاربری کانال: ")
                channel = self.get_channel_info(username)
                if channel:
                    report = self.report_generator.generate_report(channel, 'standard')
                    print("\n" + "="*70)
                    print(report)
                    print("="*70)
                else:
                    print(f"❌ کانال @{username} در کش موجود نیست. ابتدا اسکن کنید.")
                    
            elif choice == '5':
                username = input("نام کاربری کانال: ")
                channel = self.get_channel_info(username)
                if channel:
                    email_report = self.report_generator.generate_report(channel, 'email')
                    print("\n📧 گزارش ایمیل:")
                    print(email_report)
                    save = input("ذخیره در فایل؟ (y/n): ")
                    if save.lower() == 'y':
                        filename = f"email_report_{username}_{datetime.now().strftime('%Y%m%d_%H%M%S')}.txt"
                        with open(filename, 'w', encoding='utf-8') as f:
                            f.write(email_report)
                        print(f"✅ ذخیره شد: {filename}")
                else:
                    print(f"❌ کانال @{username} موجود نیست")
                    
            elif choice == '6':
                stats = self.db_manager.get_statistics()
                print("\n📈 آمار کلی:")
                print(f"   • کانال‌های اسکن شده: {stats['total_channels']}")
                print(f"   • کل تخلفات: {stats['total_violations']}")
                
            elif choice == '7':
                print("\n🔄 شروع اسکن خودکار (هر ۳ دقیقه)...")
                print("برای توقف Ctrl+C را بزنید")
                try:
                    while True:
                        await self.scan_target_channel()
                        print(f"\n⏰ اسکن بعدی در {CONFIG['SCAN_INTERVAL']} ثانیه...")
                        await asyncio.sleep(CONFIG['SCAN_INTERVAL'])
                except KeyboardInterrupt:
                    print("\n⏹️ اسکن خودکار متوقف شد")
                    
            elif choice == '8':
                username = input("نام کاربری کانال: ")
                channel = self.get_channel_info(username)
                if channel:
                    json_data = json.dumps(channel.to_dict(), ensure_ascii=False, indent=2)
                    filename = f"json_report_{username}_{datetime.now().strftime('%Y%m%d_%H%M%S')}.json"
                    with open(filename, 'w', encoding='utf-8') as f:
                        f.write(json_data)
                    print(f"✅ خروجی JSON در {filename} ذخیره شد")
                else:
                    print(f"❌ کانال @{username} موجود نیست")
                    
            elif choice == '9':
                self.channels.clear()
                print("✅ کش پاک شد")
                
            elif choice == '10':
                print("👋 خداحافظ!")
                break
                
            else:
                print("❌ انتخاب نامعتبر")

# =================================================================================
# Main Entry Point
# =================================================================================
async def main():
    """Main function"""
    scanner = AIScannerPro()
    try:
        await scanner.initialize()
        await scanner.interactive_menu()
    except Exception as e:
        logger.error(f"❌ Fatal error: {e}")
        return

if __name__ == "__main__":
    try:
        asyncio.run(main())
    except KeyboardInterrupt:
        print("\n👋 Program terminated by user")
    except Exception as e:
        print(f"❌ Error: {e}")
