
# Basic structure:

1. Introduction (OVERVIEW)
    1.1 Présentation du Projet

2. Expression des Besoins (ANALYSE)
    2.1 Model de spécification
        2.1.1 Acteurs
        2.1.2 Besoins Fonctionnels
        2.1.3 Besoins non Fonctionnels
    2.2 Model de cas d’utilisation
        2.2.1 Diagramme de cas d’utilisation
        2.2.2 Fiche descriptives
        2.2.3 Diagrammes de séquence système

3. Analyse et Conception (CONCEPTION)
    3.1 Diagrammes de classes
        3.1.1 Diagrammes de classes entités
        3.1.2 Diagrammes de classes contrôle
        3.1.3 Diagrammes de classes participants
    3.2 Diagrammes de séquence détaillé
    3.3 Diagramme de base de données

# Project ramble:
{
    ENTITIES: UNIVERSITY, VISITOR, TURNSTILE, ENTRY, QR_CODE, MEMBER (University Member), ALERT, ALERT_TYPE,
              PRINTER, AGENT, SECURITY_AGENT, TECHNICAL_AGENT, PASSAGE (Entrance/Exiting, props:date,type,visitorId, turnstileId), 
}

The university Visitor comes in contact with one of the many Entries of the university, each Entry has a
number of Turnstiles, some of these turnstiles are for entry, some are for exit. The visitor swipes his
card/ or the QR CODE that's in his university app for the turnstile to rotate, if the code is valid, then
he's allowed entry, otherwise it's blocked and the Technical-Agent is notified of this event from inside
the Watch room. The Technical-Agent tells the Security-Agent the turnstile number where there's a problem. The Security-Agent then heads towards the visitor to remove them and tell them that they need a code to enter the premise. The visitor then leaves.

If the visitor completely forgets about his card, and also forgets his phone, or doesn't have the university app installed, then he heads to the watch room to request entry. The Technical-Agent then asks him for a piece of identity to verify it against the database, if the visitor indeed exists as a UniversityMember(staff | teacher | student | intern) then the Technical-Agent prints the visitor's QR code in a small piece of paper and hands it to him so that he can enter, for this, he pays a fine of 50DA. The Technical-Agent advises them to not give his code to anyone, and to destroy the paper as soon as they get their card back or activate their app.

When the visitor exits, they simply need to swipe their card at the Exit Turnstile.

There's a flaw in the system that can be taken advantage of which is: The use of the same QR code for multiple entries by different individuals. There's a chance that some individual makes a copy of his CODE
and gives it to someone else. In order to prevent this, we need to keep track of who's in and who's out
of the university. We add a property to UniversityMember: [isInside: 0, isInside is the number of times someone entered minus the number of times they exited and ideally it should be 1 after entry and 0 after exit] [allowSuspiciouspass: true|false, this property when true allows a visitor to pass even if they have a non-zero isInside number, it is then toggled to false after they enter. It also allows exit even if isInside != 1]. If someone tries to enter using a code that someone else already used (someone whose property isInside=1), then he's refused entry, and the Technical-Agent is sent a FRAUD ALERT. The Security-Agent rushes to interrogate the visitor. The visitor is then taken to the watch room and asked to present a form of identity. 
--> If the visitor's face does not match his picture in the FRAUD ALERT, then the Security-Agent removes his means of entry and removes him from the premise after he pays the fine, then the Technical-Agent sends a notification to the holder of that Code, warning him that someone else has his code.
Of course, this warning is not enough, as the Fraud can still use the code, and get away with it sometimes.
So the Visitor gets an option to change his QR CODE and deactivate his old QR CODE.
--> If the visitor's face does indeed match his picture in the FRAUD ALERT, then the Technical-Agent toggles the allowSuspiciousPass property, allowing the legitimate visitor to pass.

How to catch the FRAUD on the exit. If a visitor tries to exist and their isInside property value is greater than 1 or less then 1, then:
greater-than-one --> he's taken to the watch room, if his face matches that on the FRAUD ALERT, then he's allowed exit, and the Technical-Agent resets his isInside value to ONE.
less-than-one --> taken to the watch room, if his face matches that in the FRAUD ALERT, then he didn't enter the premise legally through the entries. His isInside score is reset to 1 and he's allowed exit after paying a fine. == If his face however doesn't match the FRAUD ALERT, he's a fraud, the Security-Agent takes away his QR CODE, and removes him from the university through the agent gate.

In case of people with disabilities they can present their card to a Security-Agent then he can swipe their card on a turnstile and rotate it manually marking an Entrance. The disabled person is then let through the Agent Gate.

On special days when the University doors are open either for registration or special events, the turnstiles rotate without the need for some QR CODE.

Police, firemen, or legal entities enter through the agent gate, and do not pass through the turnstile system.

The main flaw of the system is that two people can use the same code and the fraud might not get caught if he times his Entrances or if he plans it with the real QR CODE owner. To catch this type of fraud, one needs to use a more advanced form of authentication like Facial Recognition as is implemented in many modern schools, or straight up Biometric Recognition by using thumbprints. But overall, the system works well enough.


# Projet ramble 2:

LEGAL_ENTITY { Police, firemen, security agents }
NORMAL = Person possessing a QR Code to scan
FORGETFUL = Person only possessing identity and no QR Code

University has many entries.
An entry has many turnstiles, and an agent gateway.
Some turnstiles are for exit and some are for entry.

OLD_QR_CODE error happens when you get a row from QRCODES table where the active property is (active=false)
REPEATED_QR_CODE error happens when the isInside property of the Person is (isInside = true)

ON ENTRY:
    
    IF Today is EVENT_DAY:
        Person walks to the turnstile
        The person rotates the turnstile and goes through
        The person enters the university

    ELSE depending on (VISITOR_TYPE) {
        CASE LEGAL_ENTITY: they enter through the agent gate
        CASE NORMAL: [
            Person walks to the turnstile
            They show their code to turnstile
            
            if AUTHENTICATION succeeds: [ ENTRY ]
                the turnstile is enabled
                Student +isInside value is updated (isInside+=1)
                The person rotates the turnstile and goes through
                The person enters the university
            
            If AUTHENTICANTION fails:
                The turnstile remains disabled
                Alert is triggered
                Alert shows up in the security room monitor notifying the technical agent
                Technical agent informs the security agent about the Turnsitle number
                Security agent goes to turnstile and brings the person to the security room
                
                If the Alert is UNKNOWN_QR_CODE: [ REMOVAL ]
                    The Security agent removes Person from entry.
                
                If the Alert is OLD_QR_CODE:
                    The Technical agent asks for identity
                    Person gives the Technical agent identity
                    Technical agent compares identity with QR CODE OWNER in Alert
                    If identity does not match: [ REMOVAL ]
                        Security agent removes Person from the entry (DOXXED FRAUD)
                    If identity matches QR CODE owner: [ TEMP PASS ]
                        ---
                        (PERSON MIGHT HAVE NOT RECEIVED HIS NEW CARD YET, OR APP OUT OF SYNC)
                        ---
                        Technical agent prints the latest Person QR CODE as a temporary pass
                        Technical agent gives the QR CODE pass to Person
                        => Person performs ENTRY

                If the Alert is REPEATED_QR_CODE (Person.isInside = true):
                    Technical agent asks for identity
                    Person gives identity to Technical Agent
                    Technical agent compares identity with QR CODE OWNER in Alert
                    If identity does not match QR CODE owner: [ UPDATE + REMOVAL ]
                        Technical agent updates CODE OWNER Person QR CODE version
                        Security agent removes Person from the entry
                    If identity matches QR CODE owner: [UPDATE + RESET + TEMP PASS]
                        ---
                        (FRAUD ARRIVED EARLIER, UPDATE QR CODE FOR CODE OWNER PERSON)
                        ---
                        
                        Technical agent updates the QR CODE version of Person
                        Technical agent resets Student +isInside value (isInside = false) 
                        Technical agent prints the latest Person QR CODE as a temporary pass
                        Technical agent gives the QR CODE pass to Person
                        
                        => Person performs ENTRY
        ]

        CASE FORGETFUL: [
            Person asks technical Agent for a temporary pass
            Technical agent asks for identity
            Person gives identity to Technical agent
            Technical agent searches identity information in Univ database
            If identity not found: [ REMOVAL ]
                Security agent removes Person from entry
            If identity found: [ FINE + TEMP PASS]
                Technical agent asks Person to pay the Fine
                Person pays the Fine
                Technical agent prints the latest Person QR CODE as a temporary pass
                Technical agent gives the QR CODE pass to Person

                => Person performs ENTRY
        ]
    }

ON EXIT

    IF Today is EVENT_DAY:
        Person walks to the turnstile
        The person rotates the turnstile and goes through
        The person exits the university
     
    ELSE depending on (VISITOR_TYPE) {
        CASE LEGAL_ENTITY: they exit through the agent gate
        CASE NORMAL: [

            Person walks to the turnstile
            They show their code to turnstile

            if AUTHENTICATION succeeds: [ EXIT ]
                the turnstile is enabled
                The person rotates the turnstile and goes through
                The person exits the university
            
            If AUTHENTICANTION fails:
                The turnstile remains disabled
                Alert is triggered
                Alert shows up in the security room monitor notifying the technical agent
                Technical agent informs the security agent about the Turnsitle number
                Security agent goes to turnstile and brings the person to the security room

                If the Alert is UNKNOWN_QR_CODE: [ FINE & REMOVAL ]
                    ---
                    (PERSON GOT IN SOMEHOW, AND TRYING TO EXIT WITH RANDOM CODE) 
                    ---
                    Technical agent asks Person to pay Fine
                    Person gives Fine to technical agent
                    Security agent removes Person from entry.
                
                If the Alert is OLD_QR_CODE:
                    ---
                    1 (PERSON COULD'VE HAD HIS CODE UPDATED BECAUSE A FRAUD TRIED TO ENTER W/ CODE SAME DAY)
                    or 2 (PERSON RECEIVED NEW CODE ON ENTRY, BUT LOST IT, AND NOW IS USING HIS OLD CODE)
                    or 3 (FRAUD ENTERED WITH SM'S CODE AND IS NOW TRYING TO EXIT)
                    ---
                    Technical agent asks person for identity
                    Person gives the Technical agent identity
                    Technical agent compares identity with QR CODE OWNER in Alert
                    
                    If identity does not match: (3) [ FINE & REMOVAL ]
                        Technical agent asks Person to pay Fine
                        Person gives Fine to technical agent
                        Security agent removes Person from entry

                    If identity matches QR CODE owner: [ TEMP PASS ]
                        ---
                        (1) or (2)
                        ---
                        Technical agent prints the latest Person QR CODE as a temporary pass
                        Technical agent gives the QR CODE pass to Person

                        => Person performs EXIT
        ]
    }

# Project ramble 3:

Concepts {

    University:place {
        int id,
        string name,
        string address
    }

    Entry:place {
        int universityId,
        int id,
        string location
    }

    Turnstile:machine {
        int entryId,
        int id,
        int type (0 for exit, 1 for entry)
    }

    Person:human {
        int id,
        string name,
        date birth,
        bool isInside
    } with subclasses: Teacher, Staff, Student, Admin, Agent

    Admin {
        int id
    }

    Agent:human {
        int entryId,
        int id,
        string name
    }

    QRCode:info {
        int personId,
        int id,
        int num,
        bool active
    }
}

Types {
    daytype = { normal, event }
    errtype = { unknown_code, old_code, repeated_code }
    prstype = { normal, forgetful, authority }
    turnstype = { exit, entry }
}

Story {

    some University
    some Admin
    some []Entry entries
    some []Turnstile turnstiles
    some []Agent agents
    some []Person people

    Person::simplePass (univ: University) {
        this.rotateTurnstileAndGo()
        this.enter(univ)
    }

    day (type: daytype) {
        
        if(type is EVENT) {
            Admin.setAutoEnableAllTT(true)
            for p in people {
                p.simplePass(University)
            }
        }
        
        if(type is NORMAL) {
            Admin.setAutoEnableAllTT(false)
            for p in people {
                p.authPass(University)
            }
        }
    }
}