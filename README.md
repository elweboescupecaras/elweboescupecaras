using System;
using System.Net;
using System.Net.NetworkInformation;
using System.Threading.Tasks;

namespace NetworkScanner
{
    class Program
    {
        static async Task Main(string[] args)
        {
            // Obtener la dirección IP local y la máscara de subred
            string localIP = "";
            string subnetMask = "";
            foreach (var ip in NetworkInterface.GetAllNetworkInterfaces())
            {
                foreach (var unicast in ip.GetIPProperties().UnicastAddresses)
                {
                    if (unicast.Address.AddressFamily == System.Net.Sockets.AddressFamily.InterNetwork)
                    {
                        localIP = unicast.Address.ToString();
                        subnetMask = unicast.IPv4Mask.ToString();
                        break;
                    }
                }
            }

            // Calcular el rango de direcciones IP de la red local
            IPAddress localAddress = IPAddress.Parse(localIP);
            IPAddress maskAddress = IPAddress.Parse(subnetMask);
            byte[] localBytes = localAddress.GetAddressBytes();
            byte[] maskBytes = maskAddress.GetAddressBytes();
            byte[] networkBytes = new byte[4];
            byte[] broadcastBytes = new byte[4];
            for (int i = 0; i < 4; i++)
            {
                networkBytes[i] = (byte)(localBytes[i] & maskBytes[i]);
                broadcastBytes[i] = (byte)(networkBytes[i] | ~maskBytes[i]);
            }
            IPAddress networkAddress = new IPAddress(networkBytes);
            IPAddress broadcastAddress = new IPAddress(broadcastBytes);

            // Crear una lista de direcciones IP posibles dentro del rango
            uint networkInt = BitConverter.ToUInt32(networkBytes, 0);
            uint broadcastInt = BitConverter.ToUInt32(broadcastBytes, 0);
            uint count = broadcastInt - networkInt;
            IPAddress[] possibleAddresses = new IPAddress[count];
            for (uint i = 0; i < count; i++)
            {
                possibleAddresses[i] = new IPAddress(networkInt + i + 1);
            }

            // Enviar una solicitud de ping a cada dirección IP y mostrar los resultados
            Console.WriteLine("Escaneando la red {0} con máscara {1}", networkAddress, maskAddress);
            Ping ping = new Ping();
            foreach (var address in possibleAddresses)
            {
                PingReply reply = await ping.SendPingAsync(address);
                if (reply.Status == IPStatus.Success)
                {
                    Console.WriteLine("Dispositivo encontrado en {0}", address);
                }
                else
                {
                    Console.WriteLine("No hay respuesta de {0}", address);
                }
            }
        }
    }
}
