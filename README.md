# DAZ-SA-AdminTools
DAZ SA AdminTools based on Daone's idea and code

Add Admin Tools to your mission

=============================================================================================================================
1. Include file in your init.c or mission file

//ADMINTOOLS - GLOBAL FUNCTIONS!
#include "$CurrentDir:\\mpmissions\\dayzOffline.chernarusplus\\_MOD\\_AdminTools\\AdminMod_Class.c" //подключаем AdminTools //mod edition (w/static variables and functions)
ref AdminMod_Class AdminMod = new AdminMod_Class();
=============================================================================================================================
2. Add _CONF directory to your mission directory like mpmissions\dayzOffline.chernarusplus and create file ADMINS_LIST.lst (Admins) and ARBITRATORS_LIST.lst (for messages only) if need (case sensitive!)
Files format: UID COMMENT - like this
1234567890123456 Admin1
2234567890123456 Admin2
=============================================================================================================================
3. Add Admin tool initialization in to OnInit() function in your cusom mission class class CustomMission: MissionServer (in init.c or mission file) like this
PS please remove all unnecessary functions calls like whitelist etc

=============================================================================================================================
class CustomMission: MissionServer
{	
	override void OnInit()
	{
		super.OnInit(); //First time call native functions
		
		AdminMod.Init();
		
		//Your code etc
	}

 // native and other functions etc

}

=============================================================================================================================	
4. Add AdminTool and more functions in class CustomMission: MissionServer like this (ADMIN MOD add below native functions)
=============================================================================================================================

class CustomMission: MissionServer
{	
	override void OnInit() //see 3^
	{
		super.OnInit(); //First time call native functions
		
		AdminMod.Init();
		
		//Your code etc
	}

     // NATIVE AND OTHER FUNCTIONS ETC
 
	//ADMIN MOD --- BEGIN
	//Author: BORIZZ.K (s-platoon.ru)
	//Based on DaOne's code!
	override void OnEvent(EventType eventTypeId, Param params) 
	{
		//super.OnEvent(eventTypeId,params);
		switch(eventTypeId)
		{
			case ChatMessageEventTypeID:
			{
				ChatMessageEventParams chat_params = ChatMessageEventParams.Cast(params);
				if (chat_params.param1 == 0 && chat_params.param2 != "" && chat_params.param3 != "") //trigger only when channel is Global == 0, Player Name does not equal to null and entered command
				{
					AdminMod.OnAdminChatRequest(params);
				}
			break;
			}
			default:
			{
				super.OnEvent(eventTypeId,params);
			break;
			}
		break;
		}
	}
	
	override void TickScheduler(float timeslice)
	{
		GetGame().GetWorld().GetPlayerList(m_Players);
		if( m_Players.Count() == 0 ) return;
		for(int i = 0; i < SCHEDULER_PLAYERS_PER_TICK; i++)
		{
			if(m_currentPlayer >= m_Players.Count() )
			{
				m_currentPlayer = 0;
			}
			//PrintString(m_currentPlayer.ToString());
			PlayerBase currentPlayer = PlayerBase.Cast(m_Players.Get(m_currentPlayer));
		
			//AdminMod
			AdminMod.OnTick(currentPlayer);
			//AdminMod
			currentPlayer.OnTick();
			m_currentPlayer++;
		}
	}
		
	override void OnPreloadEvent(PlayerIdentity identity, out bool useDB, out vector pos, out float yaw, out int queueTime)
	{
		
		if (AdminMod.m_AdminsList.Count() > 0 && AdminMod.m_AdminsList.Contains(identity.GetPlainId()))
		{
			//queueTime = 1;
			queueTime = AdminMod.spawnTime;
		}
		else if (AdminMod.spawnTime && AdminMod.spawnTime > 0)
		{
			queueTime = AdminMod.spawnTime;
		}

		// native
		if (GetHive())
		{
			// Preload data on client by character from database
			useDB = true;
		}
		else
		{
			// Preload data on client without database //Вот это я не понял зачем
			useDB = false;
			pos = "1189.3 0.0 5392.48";
			yaw = 0;
		}
		// ^ native ^
	}

	//Connect players
	override void InvokeOnConnect(PlayerBase player, PlayerIdentity identity)
	{
		super.InvokeOnConnect(player, identity);
		//Call native function

		/*
		//Если админ запретил подключения новых игроков или подключения вообще дисконнектим подключающегося игрока!
		if (!AdminMod.m_connect_new_Enable || !AdminMod.m_connect_Enable)
		{
			Print ("::: init.c ::: InvokeOnConnect ::: Admin Kick new Player " + identity.GetPlainId() + " with identity " + identity.ToString());
			Server_WhiteList.KickPlayer(player, identity, 3);
			return;
		}
		*/
		
		/*
		//Если игрок в черном списке или игрока нет в белом списке будет кик и функция вернет false
		if (!Server_WhiteList.CheckWBListConnectAllow(player, identity))
		{
			return; //IF KICK
		}
		*/
		
		AdminMod.CheckPlayer(player); //Check player fo Admins or Arbitrators list, GodMode etc //Must be AdminMod_Class is ENABLED!
		AdminMod.ConnectPlayersMessage(identity); //Message on connect
	}

	//Форсированное отключение игрока
	void ForceDisconnectPlayer(PlayerIdentity identity)
	{
		GetGame().DisconnectPlayer(identity);
		Print ("::: init.c ::: Connection players DISABLED by Admin! Player disconnect: Name: " + identity.GetName() + ", UID: " + identity.GetPlainId());
		identity = NULL;
	}

	//Disconnect players
	override void InvokeOnDisconnect( PlayerBase player )
	{
		if (player && player.GetIdentity())
		{
			string playerNAME = player.GetIdentity().GetName();
			string playerUID = player.GetIdentity().GetPlainId();
			string Message = "Player " + playerNAME + " disconnected!";
			Print("::: init.c ::: InvokeOnDisconnect ::: disconnect: " + Message);
			AdminMod.DisConnectPlayersMessage(playerNAME, playerUID);
		}
		super.InvokeOnDisconnect( player );
	}

	override void OnClientDisconnectedEvent(PlayerIdentity identity, PlayerBase player, int logoutTime, bool authFailed)
	{
		//MY ----------------------------------------------------------------------------------
		if (player.GetIdentity()) // if (identity)
		{
			//bool playerConnected = false;
			string playerNAME = player.GetIdentity().GetName();
			string playerUID = player.GetIdentity().GetPlainId();
			Print("::: init.c ::: OnClientDisconnectedEvent ::: Player: " + playerNAME + ", UID: " + playerUID + " with logoutTime " + logoutTime);
		}
		string Message = "Player " + playerNAME + " disconnected!";
		bool msgSended = false;
		
		/*
		array<Man> players = new array<Man>;
		GetGame().GetPlayers( players );
		for ( int i = 0; i < players.Count(); ++i )
		{
			PlayerBase selectedPlayer = PlayerBase.Cast(players.Get(i));
			if (selectedPlayer && selectedPlayer.GetIdentity())
			{
				PlayerIdentity selectedIdentity = selectedPlayer.GetIdentity();
				if (playerNAME == selectedIdentity.GetName() && playerUID == selectedIdentity.GetPlainId())
				{
					playerConnected = true;
					break;
				}
			}
		}
		Print("::: init.c ::: OnClientDisconnectedEvent ::: Player: " + playerNAME + ", UID: " + playerUID + " => playerConnected = " + playerConnected.ToString());
		*/
		//MY ----------------------------------------------------------------------------------

		//native
		bool disconnectNow = true;
	
		// TODO: get out of vehicle
		// using database and no saving if authorization failed
		// TODO: get out of vehicle
		// using database and no saving if authorization failed
		if (GetHive() && !authFailed)
		{			
			if (player.IsAlive())
			{	
				if (!m_LogoutPlayers.Contains(player))
				{
					Print("[Logout]: New player " + identity.GetId() + " with logout time " + logoutTime.ToString());
					
					// inform client about logout time
					GetGame().SendLogoutTime(player, logoutTime);
			
					// wait for some time before logout and save
					LogoutInfo params = new LogoutInfo(GetGame().GetTime() + logoutTime * 1000, identity.GetId());
					m_LogoutPlayers.Insert(player, params);
					
					// allow reconnecting to old char
					GetGame().AddToReconnectCache(identity);
					
					// wait until logout timer runs out
					disconnectNow = false;		
				}
				return;
			}		
		}
		// ^ native ^

		//native
		if (disconnectNow)
		{
			Print("[Logout]: New player " + identity.GetId() + " with instant logout");
			
			// inform client about instant logout
			GetGame().SendLogoutTime(player, 1);
			
			PlayerDisconnected(player, identity, identity.GetId());

			//MY ----------------------------------------------------------------------------------
			if (playerNAME && playerUID)
			{
				Print("::: init.c ::: OnClientDisconnectedEvent ::: disconnect(1): " + Message);
				AdminMod.DisConnectPlayersMessage(playerNAME, playerUID);
				msgSended = true;
			}
			//MY ----------------------------------------------------------------------------------
		}
		//^ native ^
		
		//May be not work ?
		if (!identity && playerNAME && !msgSended) //Если identity = NULL (клиент отключился), получено имя игрока в начале и сообщение не отсылалось в предыдущем условии
		{
			Print("::: init.c ::: OnClientDisconnectedEvent ::: disconnect(2): " + Message);
			AdminMod.DisConnectPlayersMessage(playerNAME, playerUID);
		}
		
	}
	//ADMIN MOD --- END

 
}
=============================================================================================================================

