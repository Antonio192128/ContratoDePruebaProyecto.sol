// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "@openzeppelin/contracts/utils/math/SafeMath.sol";

contract PrestamoDeFi {
    using SafeMath for uint256;

    // Declaración de estructuras
    struct Prestamo {
        uint id;
        uint monto;
        uint plazo; // El plazo se almacena en horas.
        uint tiempoSolicitud;
        uint tiempoLimite; // Tiempo límite para reembolsar el préstamo, en segundos desde epoch.
        address prestatario; // Dirección del solicitante del préstamo.
        bool aprobado;
        bool reembolsado;
        bool liquidado;
    }

    struct Cliente {
        bool activado;
        uint saldoGarantia;
        uint[] prestamoIds;
    }

    // Declaración de variables de estado
    address public socioPrincipal;
    mapping(address => Cliente) public clientes;
    mapping(uint => Prestamo) public prestamos;
    mapping(address => bool) public empleadosPrestamista; // Mapeo para controlar los empleados prestamistas
    
    uint private nextPrestamoId = 1; // Contador para el ID del próximo préstamo

    // Eventos
    event SolicitudPrestamo(uint indexed id, address indexed prestatario, uint monto, uint plazo);
    event PrestamoAprobado(uint indexed id, address indexed prestatario, uint monto);
    event PrestamoReembolsado(uint indexed id, address indexed prestatario, uint monto);
    event GarantiaLiquidada(uint indexed id, address indexed prestatario, uint tiempoTranscurrido);

    // Modificadores
    modifier soloSocioPrincipal() {
        require(msg.sender == socioPrincipal, "Solo el socio principal puede realizar esta accion");
        _;
    }

    modifier soloEmpleadoPrestamista() {
        require(empleadosPrestamista[msg.sender], "Solo un empleado prestamista puede realizar esta accion");
        
        _;
    }

    modifier soloClienteRegistrado() {
        require(clientes[msg.sender].activado, "Cliente no registrado");
        _;
    }

    // Constructor
    constructor() {
        socioPrincipal = msg.sender;
        empleadosPrestamista[socioPrincipal] = true;
    }

    // Funciones de gestión
    function altaPrestamista(address nuevoPrestamista) public soloSocioPrincipal {
        require(!empleadosPrestamista[nuevoPrestamista], "El prestamista ya esta dado de alta");
        empleadosPrestamista[nuevoPrestamista] = true;
    }

    function altaCliente(address nuevoCliente) public soloEmpleadoPrestamista {
        require(!clientes[nuevoCliente].activado, "El cliente ya esta dado de alta");
        clientes[nuevoCliente] = Cliente(true, 0, new uint[](0));
    }

    // Funciones de negocio
    function depositarGarantia() external payable soloClienteRegistrado {
        require(msg.value > 0, "Debe enviar una cantidad positiva de Ether");
        clientes[msg.sender].saldoGarantia = clientes[msg.sender].saldoGarantia.add(msg.value);
    }

    function solicitarPrestamo(uint monto, uint plazo) external soloClienteRegistrado returns (uint) {
        Cliente storage cliente = clientes[msg.sender];
        require(cliente.saldoGarantia >= monto, "Saldo de garantia insuficiente");
        
        uint prestamoId = nextPrestamoId++;
        Prestamo memory nuevoPrestamo = Prestamo({
            id: prestamoId,
            monto: monto,
            plazo: plazo,
            tiempoSolicitud: block.timestamp,
            tiempoLimite: block.timestamp.add(plazo.mul(3600)),
            prestatario: msg.sender,
            aprobado: false,
            reembolsado: false,
            liquidado: false
        });

        prestamos[prestamoId] = nuevoPrestamo;
        cliente.prestamoIds.push(prestamoId);

        emit SolicitudPrestamo(prestamoId, msg.sender, monto, plazo);
        return prestamoId;
    }

    function aprobarPrestamo(uint id) external soloEmpleadoPrestamista {
        Prestamo storage prestamo = prestamos[id];
        require(prestamo.id != 0 && !prestamo.aprobado && prestamo.prestatario != address(0), "Prestamo no valido o ya aprobado");
        
        prestamo.aprobado = true;
        emit PrestamoAprobado(id, prestamo.prestatario, prestamo.monto);
    }

    function reembolsarPrestamo(uint id) external soloClienteRegistrado {
        Prestamo storage prestamo = prestamos[id];
        require(prestamo.prestatario == msg.sender, "Solo el prestatario puede reembolsar este prestamo");
        require(prestamo.id != 0 && prestamo.aprobado && !prestamo.reembolsado, "Prestamo no puede ser reembolsado");
        require(clientes[msg.sender].saldoGarantia >= prestamo.monto, "Saldo de garantia insuficiente");
        
        clientes[msg.sender].saldoGarantia = clientes[msg.sender].saldoGarantia.sub(prestamo.monto);
        prestamo.reembolsado = true;

        emit PrestamoReembolsado(id, msg.sender, prestamo.monto);
        payable(socioPrincipal).transfer(prestamo.monto);
    }

    function liquidarGarantia(uint id) external soloEmpleadoPrestamista {
        Prestamo storage prestamo = prestamos[id];
        require(prestamo.id != 0 && prestamo.aprobado && !prestamo.reembolsado && block.timestamp > prestamo.tiempoLimite, "No se puede liquidar la garantia");

        prestamo.liquidado = true;
        uint tiempoTranscurrido = block.timestamp - prestamo.tiempoSolicitud;

        emit GarantiaLiquidada(id, prestamo.prestatario, tiempoTranscurrido);
    }

    function obtenerDetalleDePrestamo(uint id) external view returns (uint, uint, uint, uint, uint, address, bool, bool, bool) {
        Prestamo storage prestamo = prestamos[id];
        return (
            prestamo.id, 
            prestamo.monto, 
            prestamo.plazo.mul(3600), // Convierte el plazo de horas a segundos
            prestamo.tiempoSolicitud, 
            prestamo.tiempoLimite, 
            prestamo.prestatario, 
            prestamo.aprobado, 
            prestamo.reembolsado, 
            prestamo.liquidado
        );
    }
}# ContratoDePruebaProyecto.sol
