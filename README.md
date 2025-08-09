using System;
using System.Collections.Generic;
using System.Linq;
using System.Runtime.CompilerServices;
using FishNet.Authenticating;
using FishNet.Broadcast;
using FishNet.Broadcast.Helping;
using FishNet.Component.Observing;
using FishNet.Connection;
using FishNet.Managing.Logging;
using FishNet.Managing.Predicting;
using FishNet.Managing.Timing;
using FishNet.Managing.Transporting;
using FishNet.Managing.Utility;
using FishNet.Object;
using FishNet.Serializing;
using FishNet.Transporting;
using FishNet.Transporting.Multipass;
using GameKit.Dependencies.Utilities;
using UnityEngine;
using UnityEngine.SceneManagement;

namespace FishNet.Managing.Server
{
	// Token: 0x020001B4 RID: 436
	[DisallowMultipleComponent]
	[AddComponentMenu("FishNet/Manager/ServerManager")]
	public sealed class ServerManager : MonoBehaviour
	{
		// Token: 0x06000E15 RID: 3605 RVA: 0x0002FEEC File Offset: 0x0002E0EC
		public void RegisterBroadcast<T>(Action<NetworkConnection, T, Channel> handler, bool requireAuthentication = true) where T : struct, IBroadcast
		{
			if (handler == null)
			{
				this.NetworkManager.LogError("Broadcast cannot be registered because handler is null. This may occur when trying to register to objects which require initialization, such as events.");
				return;
			}
			ushort key = BroadcastExtensions.GetKey<T>();
			BroadcastHandlerBase broadcastHandlerBase;
			if (!this._broadcastHandlers.TryGetValueIL2CPP(key, out broadcastHandlerBase))
			{
				broadcastHandlerBase = new ClientBroadcastHandler<T>(requireAuthentication);
				this._broadcastHandlers.Add(key, broadcastHandlerBase);
			}
			broadcastHandlerBase.RegisterHandler(handler);
		}

		// Token: 0x06000E16 RID: 3606 RVA: 0x0002FF40 File Offset: 0x0002E140
		public void UnregisterBroadcast<T>(Action<NetworkConnection, T, Channel> handler) where T : struct, IBroadcast
		{
			ushort key = BroadcastExtensions.GetKey<T>();
			BroadcastHandlerBase broadcastHandlerBase;
			if (this._broadcastHandlers.TryGetValueIL2CPP(key, out broadcastHandlerBase))
			{
				broadcastHandlerBase.UnregisterHandler(handler);
			}
		}

		// Token: 0x06000E17 RID: 3607 RVA: 0x0002FF6C File Offset: 0x0002E16C
		private void ParseBroadcast(PooledReader reader, NetworkConnection conn, Channel channel)
		{
			ushort key = reader.ReadUInt16();
			int packetLength = Packets.GetPacketLength(12, reader, channel);
			BroadcastHandlerBase broadcastHandlerBase;
			if (!this._broadcastHandlers.TryGetValueIL2CPP(key, out broadcastHandlerBase))
			{
				reader.Skip(packetLength);
				return;
			}
			if (broadcastHandlerBase.RequireAuthentication && !conn.IsAuthenticated)
			{
				conn.Kick(KickReason.ExploitAttempt, LoggingType.Common, string.Format("ConnectionId {0} sent a broadcast which requires authentication, but client was not authenticated. Client has been disconnected.", conn.ClientId));
				return;
			}
			broadcastHandlerBase.InvokeHandlers(conn, reader, channel);
		}

		// Token: 0x06000E18 RID: 3608 RVA: 0x0002FFDC File Offset: 0x0002E1DC
		public void Broadcast<T>(NetworkConnection connection, T message, bool requireAuthenticated = true, Channel channel = Channel.Reliable) where T : struct, IBroadcast
		{
			if (!this.Started)
			{
				this.NetworkManager.LogWarning("Cannot send broadcast to client because server is not active.");
				return;
			}
			if (requireAuthenticated && !connection.IsAuthenticated)
			{
				this.NetworkManager.LogWarning("Cannot send broadcast to client because they are not authenticated.");
				return;
			}
			PooledWriter pooledWriter = WriterPool.Retrieve();
			BroadcastsSerializers.WriteBroadcast<T>(this.NetworkManager, pooledWriter, message, ref channel);
			ArraySegment<byte> arraySegment = pooledWriter.GetArraySegment();
			this.NetworkManager.TransportManager.SendToClient((byte)channel, arraySegment, connection, true, DataOrderType.Default);
			pooledWriter.Store();
		}

		// Token: 0x06000E19 RID: 3609 RVA: 0x00030058 File Offset: 0x0002E258
		public void Broadcast<T>(HashSet<NetworkConnection> connections, T message, bool requireAuthenticated = true, Channel channel = Channel.Reliable) where T : struct, IBroadcast
		{
			if (!this.Started)
			{
				this.NetworkManager.LogWarning("Cannot send broadcast to client because server is not active.");
				return;
			}
			bool flag = false;
			PooledWriter pooledWriter = WriterPool.Retrieve();
			BroadcastsSerializers.WriteBroadcast<T>(this.NetworkManager, pooledWriter, message, ref channel);
			ArraySegment<byte> arraySegment = pooledWriter.GetArraySegment();
			foreach (NetworkConnection networkConnection in connections)
			{
				if (requireAuthenticated && !networkConnection.IsAuthenticated)
				{
					flag = true;
				}
				else
				{
					this.NetworkManager.TransportManager.SendToClient((byte)channel, arraySegment, networkConnection, true, DataOrderType.Default);
				}
			}
			pooledWriter.Store();
			if (flag)
			{
				this.NetworkManager.LogWarning("One or more broadcast did not send to a client because they were not authenticated.");
				return;
			}
		}

		// Token: 0x06000E1A RID: 3610 RVA: 0x0003011C File Offset: 0x0002E31C
		public void BroadcastExcept<T>(HashSet<NetworkConnection> connections, NetworkConnection excludedConnection, T message, bool requireAuthenticated = true, Channel channel = Channel.Reliable) where T : struct, IBroadcast
		{
			if (!this.Started)
			{
				this.NetworkManager.LogWarning("Cannot send broadcast to client because server is not active.");
				return;
			}
			if (excludedConnection == null || !excludedConnection.IsValid)
			{
				this.Broadcast<T>(connections, message, requireAuthenticated, channel);
				return;
			}
			connections.Remove(excludedConnection);
			this.Broadcast<T>(connections, message, requireAuthenticated, channel);
		}

		// Token: 0x06000E1B RID: 3611 RVA: 0x00030174 File Offset: 0x0002E374
		public void BroadcastExcept<T>(HashSet<NetworkConnection> connections, HashSet<NetworkConnection> excludedConnections, T message, bool requireAuthenticated = true, Channel channel = Channel.Reliable) where T : struct, IBroadcast
		{
			if (!this.Started)
			{
				this.NetworkManager.LogWarning("Cannot send broadcast to client because server is not active.");
				return;
			}
			if (excludedConnections == null || excludedConnections.Count == 0)
			{
				this.Broadcast<T>(connections, message, requireAuthenticated, channel);
				return;
			}
			foreach (NetworkConnection item in excludedConnections)
			{
				connections.Remove(item);
			}
			this.Broadcast<T>(connections, message, requireAuthenticated, channel);
		}

		// Token: 0x06000E1C RID: 3612 RVA: 0x00030200 File Offset: 0x0002E400
		public void BroadcastExcept<T>(NetworkConnection excludedConnection, T message, bool requireAuthenticated = true, Channel channel = Channel.Reliable) where T : struct, IBroadcast
		{
			if (!this.Started)
			{
				this.NetworkManager.LogWarning("Cannot send broadcast to client because server is not active.");
				return;
			}
			if (excludedConnection == null || !excludedConnection.IsValid)
			{
				this.Broadcast<T>(message, requireAuthenticated, channel);
				return;
			}
			this._connectionsWithoutExclusionsCache.Clear();
			foreach (NetworkConnection item in this.Clients.Values)
			{
				this._connectionsWithoutExclusionsCache.Add(item);
			}
			this._connectionsWithoutExclusionsCache.Remove(excludedConnection);
			this.Broadcast<T>(this._connectionsWithoutExclusionsCache, message, requireAuthenticated, channel);
		}

		// Token: 0x06000E1D RID: 3613 RVA: 0x000302BC File Offset: 0x0002E4BC
		public void BroadcastExcept<T>(HashSet<NetworkConnection> excludedConnections, T message, bool requireAuthenticated = true, Channel channel = Channel.Reliable) where T : struct, IBroadcast
		{
			if (!this.Started)
			{
				this.NetworkManager.LogWarning("Cannot send broadcast to client because server is not active.");
				return;
			}
			if (excludedConnections == null || excludedConnections.Count == 0)
			{
				this.Broadcast<T>(message, requireAuthenticated, channel);
				return;
			}
			this._connectionsWithoutExclusionsCache.Clear();
			foreach (NetworkConnection item in this.Clients.Values)
			{
				this._connectionsWithoutExclusionsCache.Add(item);
			}
			foreach (NetworkConnection item2 in excludedConnections)
			{
				this._connectionsWithoutExclusionsCache.Remove(item2);
			}
			this.Broadcast<T>(this._connectionsWithoutExclusionsCache, message, requireAuthenticated, channel);
		}

		// Token: 0x06000E1E RID: 3614 RVA: 0x000303A8 File Offset: 0x0002E5A8
		public void Broadcast<T>(NetworkObject networkObject, T message, bool requireAuthenticated = true, Channel channel = Channel.Reliable) where T : struct, IBroadcast
		{
			if (networkObject == null)
			{
				this.NetworkManager.LogWarning("Cannot send broadcast because networkObject is null.");
				return;
			}
			this.Broadcast<T>(networkObject.Observers, message, requireAuthenticated, channel);
		}

		// Token: 0x06000E1F RID: 3615 RVA: 0x000303D4 File Offset: 0x0002E5D4
		public void Broadcast<T>(T message, bool requireAuthenticated = true, Channel channel = Channel.Reliable) where T : struct, IBroadcast
		{
			if (!this.Started)
			{
				this.NetworkManager.LogWarning("Cannot send broadcast to client because server is not active.");
				return;
			}
			bool flag = false;
			PooledWriter pooledWriter = WriterPool.Retrieve();
			BroadcastsSerializers.WriteBroadcast<T>(this.NetworkManager, pooledWriter, message, ref channel);
			ArraySegment<byte> arraySegment = pooledWriter.GetArraySegment();
			foreach (NetworkConnection networkConnection in this.Clients.Values)
			{
				if (requireAuthenticated && !networkConnection.IsAuthenticated)
				{
					flag = true;
				}
				else
				{
					this.NetworkManager.TransportManager.SendToClient((byte)channel, arraySegment, networkConnection, true, DataOrderType.Default);
				}
			}
			pooledWriter.Store();
			if (flag)
			{
				this.NetworkManager.LogWarning("One or more broadcast did not send to a client because they were not authenticated.");
				return;
			}
		}

		// Token: 0x1400003A RID: 58
		// (add) Token: 0x06000E20 RID: 3616 RVA: 0x000304A0 File Offset: 0x0002E6A0
		// (remove) Token: 0x06000E21 RID: 3617 RVA: 0x000304D8 File Offset: 0x0002E6D8
		public event Action<ServerConnectionStateArgs> OnServerConnectionState;

		// Token: 0x1400003B RID: 59
		// (add) Token: 0x06000E22 RID: 3618 RVA: 0x00030510 File Offset: 0x0002E710
		// (remove) Token: 0x06000E23 RID: 3619 RVA: 0x00030548 File Offset: 0x0002E748
		public event Action<NetworkConnection, bool> OnAuthenticationResult;

		// Token: 0x1400003C RID: 60
		// (add) Token: 0x06000E24 RID: 3620 RVA: 0x00030580 File Offset: 0x0002E780
		// (remove) Token: 0x06000E25 RID: 3621 RVA: 0x000305B8 File Offset: 0x0002E7B8
		public event Action<NetworkConnection, RemoteConnectionStateArgs> OnRemoteConnectionState;

		// Token: 0x17000149 RID: 329
		// (get) Token: 0x06000E26 RID: 3622 RVA: 0x000305ED File Offset: 0x0002E7ED
		// (set) Token: 0x06000E27 RID: 3623 RVA: 0x000305F5 File Offset: 0x0002E7F5
		public bool Started { get; private set; }

		// Token: 0x1700014A RID: 330
		// (get) Token: 0x06000E28 RID: 3624 RVA: 0x000305FE File Offset: 0x0002E7FE
		// (set) Token: 0x06000E29 RID: 3625 RVA: 0x00030606 File Offset: 0x0002E806
		public ServerObjects Objects { get; private set; }

		// Token: 0x1700014B RID: 331
		// (get) Token: 0x06000E2A RID: 3626 RVA: 0x0003060F File Offset: 0x0002E80F
		// (set) Token: 0x06000E2B RID: 3627 RVA: 0x00030617 File Offset: 0x0002E817
		[HideInInspector]
		public NetworkManager NetworkManager { get; private set; }

		// Token: 0x06000E2C RID: 3628 RVA: 0x00030620 File Offset: 0x0002E820
		public Authenticator GetAuthenticator()
		{
			return this._authenticator;
		}

		// Token: 0x06000E2D RID: 3629 RVA: 0x00030628 File Offset: 0x0002E828
		public void SetAuthenticator(Authenticator value)
		{
			this._authenticator = value;
			this.InitializeAuthenticator();
		}

		// Token: 0x06000E2E RID: 3630 RVA: 0x00030637 File Offset: 0x0002E837
		public void SetRemoteClientTimeout(RemoteTimeoutType timeoutType, ushort duration)
		{
			this._remoteClientTimeout = timeoutType;
			duration = (ushort)Mathf.Clamp((int)duration, 1, 1500);
			this._remoteClientTimeoutDuration = duration;
		}

		// Token: 0x06000E2F RID: 3631 RVA: 0x00030656 File Offset: 0x0002E856
		internal bool GetAllowPredictedSpawning()
		{
			return this._allowPredictedSpawning;
		}

		// Token: 0x06000E30 RID: 3632 RVA: 0x0003065E File Offset: 0x0002E85E
		internal byte GetReservedObjectIds()
		{
			return this._reservedObjectIds;
		}

		// Token: 0x06000E31 RID: 3633 RVA: 0x00030666 File Offset: 0x0002E866
		internal float GetSyncTypeRate()
		{
			return this._syncTypeRate;
		}

		// Token: 0x1700014C RID: 332
		// (get) Token: 0x06000E32 RID: 3634 RVA: 0x0003066E File Offset: 0x0002E86E
		internal ushort FrameRate
		{
			get
			{
				if (!this._changeFrameRate)
				{
					return 0;
				}
				return this._frameRate;
			}
		}

		// Token: 0x06000E33 RID: 3635 RVA: 0x00030680 File Offset: 0x0002E880
		public void SetFrameRate(ushort value)
		{
			this._frameRate = (ushort)Mathf.Clamp((int)value, 0, 500);
			this._changeFrameRate = true;
			if (this.NetworkManager != null)
			{
				this.NetworkManager.UpdateFramerate();
			}
		}

		// Token: 0x1700014D RID: 333
		// (get) Token: 0x06000E34 RID: 3636 RVA: 0x000306B5 File Offset: 0x0002E8B5
		public bool ShareIds
		{
			get
			{
				return this._shareIds;
			}
		}

		// Token: 0x06000E35 RID: 3637 RVA: 0x000306BD File Offset: 0x0002E8BD
		public bool GetStartOnHeadless()
		{
			return this._startOnHeadless;
		}

		// Token: 0x06000E36 RID: 3638 RVA: 0x000306C5 File Offset: 0x0002E8C5
		public void SetStartOnHeadless(bool value)
		{
			this._startOnHeadless = value;
		}

		// Token: 0x06000E37 RID: 3639 RVA: 0x000306CE File Offset: 0x0002E8CE
		private void OnDestroy()
		{
			ServerObjects objects = this.Objects;
			if (objects == null)
			{
				return;
			}
			objects.SubscribeToSceneLoaded(false);
		}

		// Token: 0x06000E38 RID: 3640 RVA: 0x000306E4 File Offset: 0x0002E8E4
		internal void InitializeOnce_Internal(NetworkManager manager)
		{
			this.NetworkManager = manager;
			this.Objects = new ServerObjects(manager);
			this.Objects.SubscribeToSceneLoaded(true);
			this.InitializeRpcLinks();
			this.SubscribeToTransport(false);
			this.SubscribeToTransport(true);
			this.NetworkManager.ClientManager.OnClientConnectionState += this.ClientManager_OnClientConnectionState;
			this.NetworkManager.SceneManager.OnClientLoadedStartScenes += this.SceneManager_OnClientLoadedStartScenes;
			this.NetworkManager.TimeManager.OnPostTick += this.TimeManager_OnPostTick;
			if (this._authenticator == null)
			{
				this._authenticator = base.GetComponent<Authenticator>();
			}
			if (this._authenticator != null)
			{
				this.InitializeAuthenticator();
			}
		}

		// Token: 0x06000E39 RID: 3641 RVA: 0x000307A8 File Offset: 0x0002E9A8
		private void InitializeAuthenticator()
		{
			Authenticator authenticator = this.GetAuthenticator();
			if (authenticator == null || authenticator.Initialized)
			{
				return;
			}
			if (this.NetworkManager == null)
			{
				return;
			}
			authenticator.InitializeOnce(this.NetworkManager);
			authenticator.OnAuthenticationResult += this._authenticator_OnAuthenticationResult;
		}

		// Token: 0x06000E3A RID: 3642 RVA: 0x000307FB File Offset: 0x0002E9FB
		internal void StartForHeadless()
		{
			this.GetStartOnHeadless();
		}

		// Token: 0x06000E3B RID: 3643 RVA: 0x00030804 File Offset: 0x0002EA04
		public bool StopConnection(bool sendDisconnectMessage)
		{
			if (sendDisconnectMessage)
			{
				this.SendDisconnectMessages(this.Clients.Values.ToList<NetworkConnection>(), true);
			}
			return this.NetworkManager.TransportManager.Transport.StopConnection(true);
		}

		// Token: 0x06000E3C RID: 3644 RVA: 0x00030838 File Offset: 0x0002EA38
		internal void SendDisconnectMessages(int[] connectionIds)
		{
			List<NetworkConnection> list = new List<NetworkConnection>();
			foreach (int key in connectionIds)
			{
				NetworkConnection item;
				if (this.Clients.TryGetValueIL2CPP(key, out item))
				{
					list.Add(item);
				}
			}
			if (list.Count > 0)
			{
				this.SendDisconnectMessages(list, false);
			}
		}

		// Token: 0x06000E3D RID: 3645 RVA: 0x00030888 File Offset: 0x0002EA88
		private void SendDisconnectMessages(List<NetworkConnection> conns, bool iterate)
		{
			PooledWriter pooledWriter = WriterPool.Retrieve();
			pooledWriter.WritePacketIdUnpacked(PacketId.Disconnect);
			ArraySegment<byte> arraySegment = pooledWriter.GetArraySegment();
			foreach (NetworkConnection networkConnection in conns)
			{
				networkConnection.SendToClient(0, arraySegment, false, DataOrderType.Default);
			}
			pooledWriter.Store();
			if (iterate)
			{
				this.NetworkManager.TransportManager.IterateOutgoing(true);
			}
		}

		// Token: 0x06000E3E RID: 3646 RVA: 0x00030908 File Offset: 0x0002EB08
		public bool StartConnection()
		{
			return this.NetworkManager.TransportManager.Transport.StartConnection(true);
		}

		// Token: 0x06000E3F RID: 3647 RVA: 0x00030920 File Offset: 0x0002EB20
		public bool StartConnection(ushort port)
		{
			Transport transport = this.NetworkManager.TransportManager.Transport;
			transport.SetPort(port);
			return transport.StartConnection(true);
		}

		// Token: 0x06000E40 RID: 3648 RVA: 0x00030940 File Offset: 0x0002EB40
		private void CheckClientTimeout()
		{
			if (this._remoteClientTimeout == RemoteTimeoutType.Disabled)
			{
				return;
			}
			if (this.NetworkManager.SceneManager.IsIteratingQueue(2f))
			{
				return;
			}
			float unscaledTime = Time.unscaledTime;
			if (unscaledTime < this._nextTimeoutCheckTime)
			{
				return;
			}
			this._nextTimeoutCheckTime = unscaledTime + 0.2f;
			int count = this.Clients.Count;
			if (count == 0)
			{
				return;
			}
			if (this._nextClientTimeoutCheckIndex >= count)
			{
				this._nextClientTimeoutCheckIndex = 0;
			}
			uint num = this.NetworkManager.TimeManager.TimeToTicks((double)this._remoteClientTimeoutDuration, TickRounding.RoundUp);
			int num2 = Mathf.CeilToInt(10f);
			int num3 = Mathf.Max(count / num2, 1);
			uint localTick = this.NetworkManager.TimeManager.LocalTick;
			for (int i = 0; i < num3; i++)
			{
				if (this._nextClientTimeoutCheckIndex >= this._clientsList.Count)
				{
					this._nextClientTimeoutCheckIndex = 0;
				}
				NetworkConnection networkConnection = this._clientsList[this._nextClientTimeoutCheckIndex];
				uint num4 = networkConnection.PacketTick.LocalTick;
				if (num4 == 0U)
				{
					num4 = networkConnection.ServerConnectionTick;
				}
				if (localTick - num4 >= num)
				{
					networkConnection.Kick(KickReason.UnexpectedProblem, LoggingType.Common, networkConnection.ToString() + " has timed out. You can modify this feature on the ServerManager component.");
				}
				this._nextClientTimeoutCheckIndex++;
			}
		}

		// Token: 0x06000E41 RID: 3649 RVA: 0x00030A78 File Offset: 0x0002EC78
		private void TimeManager_OnPostTick()
		{
			this.CheckClientTimeout();
		}

		// Token: 0x06000E42 RID: 3650 RVA: 0x00030A80 File Offset: 0x0002EC80
		private void ClientManager_OnClientConnectionState(ClientConnectionStateArgs obj)
		{
			if (obj.ConnectionState != LocalConnectionState.Started)
			{
				this.Objects.DestroyPending();
			}
		}

		// Token: 0x06000E43 RID: 3651 RVA: 0x00030A98 File Offset: 0x0002EC98
		private void SceneManager_OnClientLoadedStartScenes(NetworkConnection conn, bool asServer)
		{
			if (asServer)
			{
				this.Objects.RebuildObservers(conn, false);
				if (conn.IsLocalClient)
				{
					foreach (NetworkObject networkObject in this.Objects.Spawned.Values)
					{
						if (!networkObject.Observers.Contains(conn))
						{
							networkObject.SetRenderersVisible(false, false);
						}
					}
				}
			}
		}

		// Token: 0x06000E44 RID: 3652 RVA: 0x00030B1C File Offset: 0x0002ED1C
		private void SubscribeToTransport(bool subscribe)
		{
			if (this.NetworkManager == null || this.NetworkManager.TransportManager == null || this.NetworkManager.TransportManager.Transport == null)
			{
				return;
			}
			if (subscribe)
			{
				this.NetworkManager.TransportManager.Transport.OnServerReceivedData += this.Transport_OnServerReceivedData;
				this.NetworkManager.TransportManager.Transport.OnServerConnectionState += this.Transport_OnServerConnectionState;
				this.NetworkManager.TransportManager.Transport.OnRemoteConnectionState += this.Transport_OnRemoteConnectionState;
				return;
			}
			this.NetworkManager.TransportManager.Transport.OnServerReceivedData -= this.Transport_OnServerReceivedData;
			this.NetworkManager.TransportManager.Transport.OnServerConnectionState -= this.Transport_OnServerConnectionState;
			this.NetworkManager.TransportManager.Transport.OnRemoteConnectionState -= this.Transport_OnRemoteConnectionState;
		}

		// Token: 0x06000E45 RID: 3653 RVA: 0x00030C2D File Offset: 0x0002EE2D
		private void _authenticator_OnAuthenticationResult(NetworkConnection conn, bool authenticated)
		{
			if (!authenticated)
			{
				conn.Disconnect(false);
				return;
			}
			this.ClientAuthenticated(conn);
		}

		// Token: 0x06000E46 RID: 3654 RVA: 0x00030C44 File Offset: 0x0002EE44
		private void Transport_OnServerConnectionState(ServerConnectionStateArgs args)
		{
			this.Started = this.AnyServerStarted(null);
			this.NetworkManager.ClientManager.Objects.OnServerConnectionState(args);
			if (!this.Started)
			{
				MatchCondition.StoreCollections(this.NetworkManager);
				this.Objects.DespawnWithoutSynchronization(true);
			}
			this.Objects.OnServerConnectionState(args);
			LocalConnectionState connectionState = args.ConnectionState;
			if (this.NetworkManager.CanLog(LoggingType.Common))
			{
				Transport transport = this.NetworkManager.TransportManager.GetTransport(args.TransportIndex);
				string text = (transport == null) ? "Unknown" : transport.GetType().Name;
				string text2 = string.Empty;
				if (connectionState == LocalConnectionState.Starting)
				{
					text2 = string.Format(" Listening on port {0}.", transport.GetPort());
				}
				NetworkManagerExtensions.Log(string.Concat(new string[]
				{
					"Local server is ",
					connectionState.ToString().ToLower(),
					" for ",
					text,
					".",
					text2
				}));
			}
			this.NetworkManager.UpdateFramerate();
			Action<ServerConnectionStateArgs> onServerConnectionState = this.OnServerConnectionState;
			if (onServerConnectionState == null)
			{
				return;
			}
			onServerConnectionState(args);
		}

		// Token: 0x06000E47 RID: 3655 RVA: 0x00030D78 File Offset: 0x0002EF78
		private void ParseVersion(PooledReader reader, NetworkConnection conn, int transportId)
		{
			if (conn.HasSentVersion)
			{
				conn.Kick(reader, KickReason.ExploitAttempt, LoggingType.Common, "Connection " + conn.ToString() + " has sent their FishNet version after being authenticated; this is not possible under normal conditions.");
				return;
			}
			conn.HasSentVersion = true;
			string text = reader.ReadString();
			if (!(text == "4.4.5"))
			{
				conn.Kick(reader, KickReason.UnexpectedProblem, LoggingType.Warning, string.Concat(new string[]
				{
					"Connection ",
					conn.ToString(),
					" has been kicked for being on FishNet version ",
					text,
					". Server version is 4.4.5."
				}));
				return;
			}
			bool value = false;
			PooledWriter pooledWriter = WriterPool.Retrieve();
			pooledWriter.WritePacketIdUnpacked(PacketId.Version);
			pooledWriter.WriteBoolean(value);
			conn.SendToClient(0, pooledWriter.GetArraySegment(), false, DataOrderType.Default);
			WriterPool.Store(pooledWriter);
			Authenticator authenticator = this.GetAuthenticator();
			if (authenticator != null && !this.NetworkManager.TransportManager.IsLocalTransport(transportId, -1))
			{
				authenticator.OnRemoteConnection(conn);
				return;
			}
			this.ClientAuthenticated(conn);
		}

		// Token: 0x06000E48 RID: 3656 RVA: 0x00030E60 File Offset: 0x0002F060
		private void Transport_OnRemoteConnectionState(RemoteConnectionStateArgs args)
		{
			int connectionId = args.ConnectionId;
			if (connectionId < 0 || connectionId > 2147483647)
			{
				this.Kick(args.ConnectionId, KickReason.UnexpectedProblem, LoggingType.Error, string.Format("The transport you are using supplied an invalid connection Id of {0}. Connection Id values must range between 0 and {1}. The client has been disconnected.", connectionId, int.MaxValue));
				return;
			}
			if (args.ConnectionState != RemoteConnectionState.Started)
			{
				NetworkConnection networkConnection;
				if (args.ConnectionState == RemoteConnectionState.Stopped && this.Clients.TryGetValueIL2CPP(connectionId, out networkConnection))
				{
					networkConnection.SetDisconnecting(true);
					Action<NetworkConnection, RemoteConnectionStateArgs> onRemoteConnectionState = this.OnRemoteConnectionState;
					if (onRemoteConnectionState != null)
					{
						onRemoteConnectionState(networkConnection, args);
					}
					this.Clients.Remove(connectionId);
					this._clientsList.Remove(networkConnection);
					this.Objects.ClientDisconnected(networkConnection);
					this.BroadcastClientConnectionChange(false, networkConnection);
					Queue<int> predictedObjectIds = networkConnection.PredictedObjectIds;
					while (predictedObjectIds.Count > 0)
					{
						this.Objects.CacheObjectId(predictedObjectIds.Dequeue());
					}
					networkConnection.ResetState();
					this.NetworkManager.Log(string.Format("Remote connection stopped for Id {0}.", connectionId));
				}
				return;
			}
			this.NetworkManager.Log(string.Format("Remote connection started for Id {0}.", connectionId));
			NetworkConnection networkConnection2 = new NetworkConnection(this.NetworkManager, connectionId, args.TransportIndex, true);
			this.Clients.Add(args.ConnectionId, networkConnection2);
			this._clientsList.Add(networkConnection2);
			Action<NetworkConnection, RemoteConnectionStateArgs> onRemoteConnectionState2 = this.OnRemoteConnectionState;
			if (onRemoteConnectionState2 == null)
			{
				return;
			}
			onRemoteConnectionState2(networkConnection2, args);
		}

		// Token: 0x06000E49 RID: 3657 RVA: 0x00030FBC File Offset: 0x0002F1BC
		private void SendAuthenticated(NetworkConnection conn)
		{
			PooledWriter pooledWriter = WriterPool.Retrieve();
			pooledWriter.WritePacketIdUnpacked(PacketId.Authenticated);
			pooledWriter.WriteNetworkConnection(conn);
			PredictionManager predictionManager = this.NetworkManager.PredictionManager;
			if (this.GetAllowPredictedSpawning())
			{
				int num = Mathf.Min(this.Objects.GetObjectIdCache().Count, (int)this.GetReservedObjectIds());
				pooledWriter.WriteUInt8Unpacked((byte)num);
				for (int i = 0; i < num; i++)
				{
					ushort num2 = (ushort)this.Objects.GetNextNetworkObjectId(false);
					pooledWriter.WriteNetworkObjectId((int)num2);
					conn.PredictedObjectIds.Enqueue((int)num2);
				}
			}
			this.NetworkManager.TransportManager.SendToClient(0, pooledWriter.GetArraySegment(), conn, true, DataOrderType.Default);
			pooledWriter.Store();
		}

		// Token: 0x06000E4A RID: 3658 RVA: 0x00031062 File Offset: 0x0002F262
		private void Transport_OnServerReceivedData(ServerReceivedDataArgs args)
		{
			this.ParseReceived(args);
		}

		// Token: 0x06000E4B RID: 3659 RVA: 0x0003106C File Offset: 0x0002F26C
		private void ParseReceived(ServerReceivedDataArgs args)
		{
			ServerManager.<>c__DisplayClass84_0 CS$<>8__locals1;
			CS$<>8__locals1.<>4__this = this;
			CS$<>8__locals1.args = args;
			if (CS$<>8__locals1.args.ConnectionId < 0)
			{
				return;
			}
			ArraySegment<byte> segment;
			if (this.NetworkManager.TransportManager.HasIntermediateLayer)
			{
				segment = this.NetworkManager.TransportManager.ProcessIntermediateIncoming(CS$<>8__locals1.args.Data, false);
			}
			else
			{
				segment = CS$<>8__locals1.args.Data;
			}
			this.NetworkManager.StatisticsManager.NetworkTraffic.LocalServerReceivedData((ulong)((long)segment.Count));
			if (segment.Count <= 4)
			{
				return;
			}
			int mtu = this.NetworkManager.TransportManager.GetMTU(CS$<>8__locals1.args.TransportIndex, (byte)CS$<>8__locals1.args.Channel);
			if (segment.Count > mtu)
			{
				this.<ParseReceived>g__ExceededMTUKick|84_0(ref CS$<>8__locals1);
				return;
			}
			TimeManager timeManager = this.NetworkManager.TimeManager;
			bool hasIntermediateLayer = this.NetworkManager.TransportManager.HasIntermediateLayer;
			PacketId packetId = PacketId.Unset;
			PooledReader pooledReader = null;
			try
			{
				Reader.DataSource source = Reader.DataSource.Client;
				pooledReader = ReaderPool.Retrieve(segment, this.NetworkManager, source);
				uint num = pooledReader.ReadTickUnpacked();
				timeManager.LastPacketTick.Update(num, EstimatedTick.OldTickOption.Discard, true);
				if (pooledReader.PeekPacketId() == PacketId.Split)
				{
					pooledReader.ReadPacketId();
					int expectedMessages;
					this._splitReader.GetHeader(pooledReader, out expectedMessages);
					this._splitReader.Write(num, pooledReader, expectedMessages);
					ArraySegment<byte> fullMessage = this._splitReader.GetFullMessage();
					if (fullMessage.Count == 0)
					{
						return;
					}
					pooledReader.Initialize(fullMessage, this.NetworkManager, source);
				}
				while (pooledReader.Remaining > 0)
				{
					packetId = pooledReader.ReadPacketId();
					NetworkConnection networkConnection;
					if (!this.Clients.TryGetValueIL2CPP(CS$<>8__locals1.args.ConnectionId, out networkConnection))
					{
						this.Kick(CS$<>8__locals1.args.ConnectionId, KickReason.UnexpectedProblem, LoggingType.Error, string.Format("ConnectionId {0} not found within Clients. Connection will be kicked immediately.", CS$<>8__locals1.args.ConnectionId));
						break;
					}
					networkConnection.LocalTick.Update(timeManager, num, EstimatedTick.OldTickOption.Discard, true);
					networkConnection.PacketTick.Update(timeManager, num, EstimatedTick.OldTickOption.SetLastRemoteTick, true);
					if (!networkConnection.IsAuthenticated && packetId != PacketId.Version && packetId != PacketId.Broadcast)
					{
						networkConnection.Kick(KickReason.ExploitAttempt, LoggingType.Common, string.Format("ConnectionId {0} sent packetId {1} without being authenticated. Connection will be kicked immediately.", networkConnection.ClientId, packetId));
						break;
					}
					if (packetId == PacketId.Replicate)
					{
						this.Objects.ParseReplicateRpc(pooledReader, networkConnection, CS$<>8__locals1.args.Channel);
					}
					else if (packetId == PacketId.ServerRpc)
					{
						this.Objects.ParseServerRpc(pooledReader, networkConnection, CS$<>8__locals1.args.Channel);
					}
					else if (packetId == PacketId.ObjectSpawn)
					{
						if (!this.GetAllowPredictedSpawning())
						{
							networkConnection.Kick(KickReason.ExploitAttempt, LoggingType.Common, string.Format("ConnectionId {0} sent a predicted spawn while predicted spawning is not enabled. Connection will be kicked immediately.", networkConnection.ClientId));
							break;
						}
						this.Objects.ReadSpawn(pooledReader, networkConnection);
					}
					else if (packetId == PacketId.ObjectDespawn)
					{
						if (!this.GetAllowPredictedSpawning())
						{
							networkConnection.Kick(KickReason.ExploitAttempt, LoggingType.Common, string.Format("ConnectionId {0} sent a predicted spawn while predicted spawning is not enabled. Connection will be kicked immediately.", networkConnection.ClientId));
							break;
						}
						this.Objects.ReadDespawn(pooledReader, networkConnection);
					}
					else if (packetId == PacketId.Broadcast)
					{
						this.ParseBroadcast(pooledReader, networkConnection, CS$<>8__locals1.args.Channel);
					}
					else if (packetId == PacketId.PingPong)
					{
						this.ParsePingPong(pooledReader, networkConnection);
					}
					else
					{
						if (packetId != PacketId.Version)
						{
							this.NetworkManager.LogError(string.Format("Server received an unhandled PacketId of {0} on channel {1} from connectionId {2}. Connection will be kicked immediately.", (ushort)packetId, CS$<>8__locals1.args.Channel, CS$<>8__locals1.args.ConnectionId));
							this.NetworkManager.TransportManager.Transport.StopConnection(CS$<>8__locals1.args.ConnectionId, true);
							break;
						}
						this.ParseVersion(pooledReader, networkConnection, CS$<>8__locals1.args.TransportIndex);
					}
				}
			}
			catch (Exception ex)
			{
				this.Kick(CS$<>8__locals1.args.ConnectionId, KickReason.MalformedData, LoggingType.Error, string.Format("Server encountered an error while parsing data for packetId {0} from connectionId {1}. Connection will be kicked immediately. Message: {2}.", packetId, CS$<>8__locals1.args.ConnectionId, ex.Message));
			}
			finally
			{
				if (pooledReader != null)
				{
					pooledReader.Store();
				}
			}
		}

		// Token: 0x06000E4C RID: 3660 RVA: 0x000314B8 File Offset: 0x0002F6B8
		private void ParsePingPong(PooledReader reader, NetworkConnection conn)
		{
			uint clientTick = reader.ReadTickUnpacked();
			if (conn.CanPingPong())
			{
				this.NetworkManager.TimeManager.SendPong(conn, clientTick);
			}
		}

		// Token: 0x06000E4D RID: 3661 RVA: 0x000314E6 File Offset: 0x0002F6E6
		private void ClientAuthenticated(NetworkConnection connection)
		{
			connection.ConnectionAuthenticated();
			this.BroadcastClientConnectionChange(true, connection);
			this.SendAuthenticated(connection);
			Action<NetworkConnection, bool> onAuthenticationResult = this.OnAuthenticationResult;
			if (onAuthenticationResult != null)
			{
				onAuthenticationResult(connection, true);
			}
			this.NetworkManager.SceneManager.OnClientAuthenticated(connection);
		}

		// Token: 0x06000E4E RID: 3662 RVA: 0x00031524 File Offset: 0x0002F724
		private void BroadcastClientConnectionChange(bool connected, NetworkConnection conn)
		{
			if (!conn.IsAuthenticated)
			{
				return;
			}
			if (this.ShareIds)
			{
				ClientConnectionChangeBroadcast message = new ClientConnectionChangeBroadcast
				{
					Connected = connected,
					Id = conn.ClientId
				};
				foreach (NetworkConnection networkConnection in this.Clients.Values)
				{
					if (networkConnection.IsAuthenticated)
					{
						this.Broadcast<ClientConnectionChangeBroadcast>(networkConnection, message, true, Channel.Reliable);
					}
				}
				if (connected)
				{
					List<int> list = CollectionCaches<int>.RetrieveList();
					foreach (int item in this.Clients.Keys)
					{
						list.Add(item);
					}
					ConnectedClientsBroadcast message2 = new ConnectedClientsBroadcast
					{
						Values = list
					};
					conn.Broadcast<ConnectedClientsBroadcast>(message2, true, Channel.Reliable);
					CollectionCaches<int>.Store(list);
					return;
				}
			}
			else if (connected)
			{
				ClientConnectionChangeBroadcast message3 = new ClientConnectionChangeBroadcast
				{
					Connected = connected,
					Id = conn.ClientId
				};
				this.Broadcast<ClientConnectionChangeBroadcast>(conn, message3, true, Channel.Reliable);
			}
		}

		// Token: 0x1400003D RID: 61
		// (add) Token: 0x06000E4F RID: 3663 RVA: 0x00031668 File Offset: 0x0002F868
		// (remove) Token: 0x06000E50 RID: 3664 RVA: 0x000316A0 File Offset: 0x0002F8A0
		public event Action<NetworkConnection, int, KickReason> OnClientKick;

		// Token: 0x06000E51 RID: 3665 RVA: 0x000316D8 File Offset: 0x0002F8D8
		public bool OneServerStarted()
		{
			int num = 0;
			TransportManager transportManager = this.NetworkManager.TransportManager;
			Multipass multipass = transportManager.Transport as Multipass;
			if (multipass != null)
			{
				using (IEnumerator<Transport> enumerator = multipass.Transports.GetEnumerator())
				{
					while (enumerator.MoveNext())
					{
						if (enumerator.Current.GetConnectionState(true) == LocalConnectionState.Started)
						{
							num++;
						}
					}
					goto IL_63;
				}
			}
			if (transportManager.Transport.GetConnectionState(true) == LocalConnectionState.Started)
			{
				num = 1;
			}
			IL_63:
			return num == 1;
		}

		// Token: 0x06000E52 RID: 3666 RVA: 0x0003175C File Offset: 0x0002F95C
		public bool AnyServerStarted(int? excludedIndex = null)
		{
			TransportManager transportManager = this.NetworkManager.TransportManager;
			Multipass multipass = transportManager.Transport as Multipass;
			if (multipass != null)
			{
				Transport y = (excludedIndex == null) ? null : multipass.GetTransport(excludedIndex.Value);
				using (IEnumerator<Transport> enumerator = multipass.Transports.GetEnumerator())
				{
					while (enumerator.MoveNext())
					{
						Transport transport = enumerator.Current;
						if (!(transport == y) && transport.GetConnectionState(true) == LocalConnectionState.Started)
						{
							return true;
						}
					}
					return false;
				}
			}
			return transportManager.Transport.GetConnectionState(true) == LocalConnectionState.Started;
		}

		// Token: 0x06000E53 RID: 3667 RVA: 0x00031808 File Offset: 0x0002FA08
		public void Spawn(GameObject go, NetworkConnection ownerConnection = null, Scene scene = default(Scene))
		{
			if (go == null)
			{
				this.NetworkManager.LogWarning("GameObject cannot be spawned because it is null.");
				return;
			}
			NetworkObject component = go.GetComponent<NetworkObject>();
			this.Spawn(component, ownerConnection, scene);
		}

		// Token: 0x06000E54 RID: 3668 RVA: 0x0003183F File Offset: 0x0002FA3F
		public void Spawn(NetworkObject nob, NetworkConnection ownerConnection = null, Scene scene = default(Scene))
		{
			if (!nob.IsSpawnable)
			{
				this.NetworkManager.LogWarning(string.Format("NetworkObject {0} cannot be spawned because it is not marked as spawnable.", nob));
				return;
			}
			this.Objects.Spawn(nob, ownerConnection, scene);
		}

		// Token: 0x06000E55 RID: 3669 RVA: 0x00031870 File Offset: 0x0002FA70
		public void Despawn(GameObject go, DespawnType? despawnType = null)
		{
			if (go == null)
			{
				this.NetworkManager.LogWarning("GameObject cannot be despawned because it is null.");
				return;
			}
			NetworkObject component = go.GetComponent<NetworkObject>();
			this.Despawn(component, despawnType);
		}

		// Token: 0x06000E56 RID: 3670 RVA: 0x000318A8 File Offset: 0x0002FAA8
		public void Despawn(NetworkObject networkObject, DespawnType? despawnType = null)
		{
			DespawnType despawnType2 = (despawnType == null) ? networkObject.GetDefaultDespawnType() : despawnType.Value;
			this.Objects.Despawn(networkObject, despawnType2, true);
		}

		// Token: 0x06000E57 RID: 3671 RVA: 0x000318DC File Offset: 0x0002FADC
		public void Kick(NetworkConnection conn, KickReason kickReason, LoggingType loggingType = LoggingType.Common, string log = "")
		{
			if (!conn.IsValid)
			{
				return;
			}
			Action<NetworkConnection, int, KickReason> onClientKick = this.OnClientKick;
			if (onClientKick != null)
			{
				onClientKick(conn, conn.ClientId, kickReason);
			}
			if (conn.IsActive)
			{
				conn.Disconnect(true);
			}
			if (!string.IsNullOrEmpty(log))
			{
				this.NetworkManager.Log(loggingType, log);
			}
		}

		// Token: 0x06000E58 RID: 3672 RVA: 0x00031934 File Offset: 0x0002FB34
		public void Kick(int clientId, KickReason kickReason, LoggingType loggingType = LoggingType.Common, string log = "")
		{
			Action<NetworkConnection, int, KickReason> onClientKick = this.OnClientKick;
			if (onClientKick != null)
			{
				onClientKick(null, clientId, kickReason);
			}
			this.NetworkManager.TransportManager.Transport.StopConnection(clientId, true);
			if (!string.IsNullOrEmpty(log))
			{
				this.NetworkManager.Log(loggingType, log);
			}
		}

		// Token: 0x06000E59 RID: 3673 RVA: 0x00031984 File Offset: 0x0002FB84
		public void Kick(NetworkConnection conn, Reader reader, KickReason kickReason, LoggingType loggingType = LoggingType.Common, string log = "")
		{
			reader.Clear();
			this.Kick(conn, kickReason, loggingType, log);
		}

		// Token: 0x06000E5A RID: 3674 RVA: 0x00031998 File Offset: 0x0002FB98
		private void InitializeRpcLinks()
		{
			ushort startingRpcLinkIndex = NetworkManager.StartingRpcLinkIndex;
			for (ushort num = ushort.MaxValue; num >= startingRpcLinkIndex; num -= 1)
			{
				this._availableRpcLinkIndexes.Enqueue(num);
			}
		}

		// Token: 0x06000E5B RID: 3675 RVA: 0x000319C8 File Offset: 0x0002FBC8
		internal bool GetRpcLink(out ushort value)
		{
			if (this._availableRpcLinkIndexes.Count > 0)
			{
				value = this._availableRpcLinkIndexes.Dequeue();
				return true;
			}
			value = 0;
			return false;
		}

		// Token: 0x06000E5C RID: 3676 RVA: 0x000319EB File Offset: 0x0002FBEB
		internal void SetRpcLink(ushort linkIndex, RpcLink data)
		{
			this.RpcLinks[linkIndex] = data;
		}

		// Token: 0x06000E5D RID: 3677 RVA: 0x000319FC File Offset: 0x0002FBFC
		internal void StoreRpcLinks(Dictionary<uint, RpcLinkType> links)
		{
			foreach (RpcLinkType rpcLinkType in links.Values)
			{
				this._availableRpcLinkIndexes.Enqueue(rpcLinkType.LinkIndex);
			}
		}

		// Token: 0x06000E5F RID: 3679 RVA: 0x00031B1E File Offset: 0x0002FD1E
		[CompilerGenerated]
		private void <ParseReceived>g__ExceededMTUKick|84_0(ref ServerManager.<>c__DisplayClass84_0 A_1)
		{
			this.Kick(A_1.args.ConnectionId, KickReason.ExploitExcessiveData, LoggingType.Common, string.Format("ConnectionId {0} sent a message larger than allowed amount. Connection will be kicked immediately.", A_1.args.ConnectionId));
		}

		// Token: 0x0400070C RID: 1804
		private readonly Dictionary<ushort, BroadcastHandlerBase> _broadcastHandlers = new Dictionary<ushort, BroadcastHandlerBase>();

		// Token: 0x0400070D RID: 1805
		private HashSet<NetworkConnection> _connectionsWithoutExclusionsCache = new HashSet<NetworkConnection>();

		// Token: 0x04000713 RID: 1811
		[HideInInspector]
		public Dictionary<int, NetworkConnection> Clients = new Dictionary<int, NetworkConnection>();

		// Token: 0x04000714 RID: 1812
		private List<NetworkConnection> _clientsList = new List<NetworkConnection>();

		// Token: 0x04000716 RID: 1814
		[Tooltip("Authenticator for this ServerManager. May be null if not using authentication.")]
		[SerializeField]
		private Authenticator _authenticator;

		// Token: 0x04000717 RID: 1815
		[Tooltip("What platforms to enable remote client timeout.")]
		[SerializeField]
		private RemoteTimeoutType _remoteClientTimeout = RemoteTimeoutType.Development;

		// Token: 0x04000718 RID: 1816
		[Tooltip("How long in seconds a client must go without sending any packets before getting disconnected. This is independent of any transport settings.")]
		[Range(1f, 1500f)]
		[SerializeField]
		private ushort _remoteClientTimeoutDuration = 60;

		// Token: 0x04000719 RID: 1817
		[Tooltip("True to allow clients to use predicted spawning. While true, each NetworkObject you wish this feature to apply towards must have a PredictedSpawn component. Predicted spawns can have custom validation on the server.")]
		[SerializeField]
		private bool _allowPredictedSpawning;

		// Token: 0x0400071A RID: 1818
		[Tooltip("Maximum number of Ids to reserve on clients for predicted spawning. Higher values will allow clients to send more predicted spawns per second but may reduce availability of ObjectIds with high player counts.")]
		[Range(1f, 100f)]
		[SerializeField]
		private byte _reservedObjectIds = 15;

		// Token: 0x0400071B RID: 1819
		[Tooltip("Default send rate for SyncTypes. A value of 0f will send changed values every tick.")]
		[Range(0f, 60f)]
		[SerializeField]
		private float _syncTypeRate = 0.1f;

		// Token: 0x0400071C RID: 1820
		[Tooltip("How to pack object spawns.")]
		[SerializeField]
		internal TransformPackingData SpawnPacking = new TransformPackingData
		{
			Position = AutoPackType.Unpacked,
			Rotation = AutoPackType.PackedLess,
			Scale = AutoPackType.PackedLess
		};

		// Token: 0x0400071D RID: 1821
		[Tooltip("True to automatically set the frame rate when the client connects.")]
		[SerializeField]
		private bool _changeFrameRate = true;

		// Token: 0x0400071E RID: 1822
		[Tooltip("Maximum frame rate the server may run at. When as host this value runs at whichever is higher between client and server.")]
		[Range(1f, 500f)]
		[SerializeField]
		private ushort _frameRate = 500;

		// Token: 0x0400071F RID: 1823
		[Tooltip("True to share the Ids of clients and the objects they own with other clients. No sensitive information is shared.")]
		[SerializeField]
		private bool _shareIds = true;

		// Token: 0x04000720 RID: 1824
		[Tooltip("True to automatically start the server connection when running as headless.")]
		[SerializeField]
		private bool _startOnHeadless = true;

		// Token: 0x04000721 RID: 1825
		private int _nextClientTimeoutCheckIndex;

		// Token: 0x04000722 RID: 1826
		private float _nextTimeoutCheckTime;

		// Token: 0x04000723 RID: 1827
		private SplitReader _splitReader = new SplitReader();

		// Token: 0x04000724 RID: 1828
		public const ushort MAXIMUM_REMOTE_CLIENT_TIMEOUT_DURATION = 1500;

		// Token: 0x04000726 RID: 1830
		internal Dictionary<ushort, RpcLink> RpcLinks = new Dictionary<ushort, RpcLink>();

		// Token: 0x04000727 RID: 1831
		private Queue<ushort> _availableRpcLinkIndexes = new Queue<ushort>();
	}
}
