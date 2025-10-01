--
-- PostgreSQL database dump
--

-- Dumped from database version 17.5
-- Dumped by pg_dump version 17.5

-- Started on 2025-09-30 21:42:09

SET statement_timeout = 0;
SET lock_timeout = 0;
SET idle_in_transaction_session_timeout = 0;
SET transaction_timeout = 0;
SET client_encoding = 'UTF8';
SET standard_conforming_strings = on;
SELECT pg_catalog.set_config('search_path', '', false);
SET check_function_bodies = false;
SET xmloption = content;
SET client_min_messages = warning;
SET row_security = off;

--
-- TOC entry 4 (class 2615 OID 2200)
-- Name: public; Type: SCHEMA; Schema: -; Owner: pg_database_owner
--

CREATE SCHEMA public;


ALTER SCHEMA public OWNER TO pg_database_owner;

--
-- TOC entry 5017 (class 0 OID 0)
-- Dependencies: 4
-- Name: SCHEMA public; Type: COMMENT; Schema: -; Owner: pg_database_owner
--

COMMENT ON SCHEMA public IS 'standard public schema';


--
-- TOC entry 259 (class 1255 OID 16985)
-- Name: anhadir_carrito(integer, integer, integer); Type: PROCEDURE; Schema: public; Owner: postgres
--

CREATE PROCEDURE public.anhadir_carrito(IN v_idcliente integer, IN v_idproducto integer, IN v_cantidad integer)
    LANGUAGE plpgsql
    AS $$
DECLARE
	v_idcarrito INTEGER;
	v_precio NUMERIC(10,2);
	v_subtotal NUMERIC(10,2);
	v_stock INTEGER;
	v_verificador INTEGER;
	v_cantidad_total INTEGER;
BEGIN
-- 2. CONSULTAR PRECIO, STOCK SEGUN EL PRODUCTO
	SELECT P.PRECIO, P.STOCK
	INTO v_precio, v_stock
	FROM PRODUCTO AS P
	WHERE P.IDPRODUCTO = v_idproducto;
-- 1. CONSULTAR IDCARRITO SEGUN IDCLIENTE
	SELECT CA.IDCARRITO
	INTO v_idcarrito
	FROM CARRITO_COMPRA AS CA
	WHERE CA.FKCLIENTE = v_idcliente;
-- Si hay registro del producto, sumar la cantidad nueva a la actual
	SELECT DC.IDDETALLECARRITO, DC.CANTIDAD
	INTO v_verificador, v_cantidad_total
	FROM DETALLE_CARRITO AS DC
	WHERE DC.FKCARRITO = v_idcarrito
	AND DC.FKPRODUCTO = v_idproducto;
-- Validar Si el usuario ya habia agregado el producto anteriormente
	IF v_verificador IS NOT NULL THEN
			-- 0. VALIDAR STOCK DISPONIBLE
			v_cantidad_total := v_cantidad_total + v_cantidad;
			IF v_cantidad_total > v_stock THEN
				RAISE EXCEPTION 'STOCK INSUFICIENTE, STOCK RESTANTE %', v_stock;
			END IF;
			-- 3. CALCULAR PRECIO SEGUN CANTIDAD
			v_subtotal := v_precio * v_cantidad_total;
			-- Actualizar 
			UPDATE DETALLE_CARRITO
			SET 
			CANTIDAD = CANTIDAD + v_cantidad, 
			SUBTOTAL = v_subtotal
			WHERE IDDETALLECARRITO = v_verificador;
		
	ELSE

			-- Si es la primera vez que agrega el producto, insertar registro
			-- 0. VALIDAR STOCK DISPONIBLE
			IF v_cantidad > v_stock THEN
				RAISE EXCEPTION 'STOCK INSUFICIENTE, STOCK RESTANTE %', v_stock;
			END IF;
			-- 3. CALCULAR PRECIO SEGUN CANTIDAD
			v_subtotal := v_precio * v_cantidad;
			-- 4. INSERTAR DETALLE_CARRITO
			INSERT INTO DETALLE_CARRITO(FKCARRITO, FKPRODUCTO, CANTIDAD, SUBTOTAL)
			VALUES(v_idcarrito, v_idproducto, v_cantidad, v_subtotal);
	END IF;
END;
$$;


ALTER PROCEDURE public.anhadir_carrito(IN v_idcliente integer, IN v_idproducto integer, IN v_cantidad integer) OWNER TO postgres;

--
-- TOC entry 257 (class 1255 OID 16978)
-- Name: f_actualizar_stock(); Type: FUNCTION; Schema: public; Owner: postgres
--

CREATE FUNCTION public.f_actualizar_stock() RETURNS trigger
    LANGUAGE plpgsql
    AS $$
BEGIN
	UPDATE PRODUCTO
	SET STOCK = STOCK - NEW.CANTIDAD
	WHERE IDPRODUCTO = NEW.FKPRODUCTO;
	RETURN NEW;
END;
$$;


ALTER FUNCTION public.f_actualizar_stock() OWNER TO postgres;

--
-- TOC entry 260 (class 1255 OID 16991)
-- Name: f_after_update_detalle_carrito(); Type: FUNCTION; Schema: public; Owner: postgres
--

CREATE FUNCTION public.f_after_update_detalle_carrito() RETURNS trigger
    LANGUAGE plpgsql
    AS $$
DECLARE
	v_subtotal NUMERIC(10,2);
BEGIN
	/*
		El Objetivo de NEW.SubTotal - OLD.SubTotal:
			Es Determinar el SubTotal del Nuevo Producto Agregado
			Para luego ser Sumado con el Total del Carrito
	*/
	v_subtotal := NEW.SubTotal - OLD.SubTotal;
	UPDATE CARRITO_COMPRA
	SET TOTAL = TOTAL + v_subtotal
	WHERE IDCARRITO = NEW.FkCarrito
	RETURNING TOTAL
	INTO v_subtotal;
	-- Validar Si es un Numero Negativo, Volver a Inicializar el total del Carrito en 0
	IF v_subtotal < 0 THEN
		UPDATE CARRITO_COMPRA
		SET TOTAL = 0
		WHERE IDCARRITO = NEW.FkCarrito;
	END IF;
	RETURN NEW;
END;
$$;


ALTER FUNCTION public.f_after_update_detalle_carrito() OWNER TO postgres;

--
-- TOC entry 256 (class 1255 OID 16983)
-- Name: f_delete_detalle_carrito(); Type: FUNCTION; Schema: public; Owner: postgres
--

CREATE FUNCTION public.f_delete_detalle_carrito() RETURNS trigger
    LANGUAGE plpgsql
    AS $$
DECLARE 
	v_idcarrito INTEGER;
	v_cantidad INTEGER;
	v_iddetallecarrito INTEGER;
BEGIN
-- SubQuery para Encontrar el Id Del Carrito segun el Id de la venta Realizada por el Cliente.
	SELECT IDCARRITO
	INTO v_idcarrito
	FROM CARRITO_COMPRA
	WHERE FKCLIENTE = (
		SELECT FKCLIENTE
		FROM VENTA
		WHERE IDVENTA = NEW.FKVENTA
	);
-- Eliminar las cantidades en detalle_carrito despues de hacer la compra
	UPDATE DETALLE_CARRITO
	SET 
	CANTIDAD = CANTIDAD - NEW.CANTIDAD,
	SUBTOTAL = SUBTOTAL - NEW.SUBTOTAL
	WHERE FKCARRITO = v_idcarrito
	AND FKPRODUCTO = NEW.FKPRODUCTO
	RETURNING CANTIDAD, IDDETALLECARRITO
	INTO v_cantidad, v_iddetallecarrito;
-- Si el resultado es 0 o negativo, eliminar completamente el registro
	IF v_cantidad <= 0 THEN
		DELETE FROM DETALLE_CARRITO
		WHERE IDDETALLECARRITO = v_iddetallecarrito;
	END IF;
	RETURN NEW;
END;
$$;


ALTER FUNCTION public.f_delete_detalle_carrito() OWNER TO postgres;

--
-- TOC entry 238 (class 1255 OID 16969)
-- Name: f_login_admin(character varying); Type: FUNCTION; Schema: public; Owner: postgres
--

CREATE FUNCTION public.f_login_admin(v_email character varying) RETURNS TABLE(idadmin integer, nombre character varying, email character varying, pwd character varying)
    LANGUAGE plpgsql
    AS $$
DECLARE
	v_verificador INTEGER;
BEGIN
-- 1. Consultar Email 
	SELECT A.IDADMIN
	INTO v_verificador
	FROM ADMINISTRADOR AS A
	WHERE A.EMAIL = v_email;
-- 2. Validar Si Existe el Cliente
	IF v_verificador IS NULL THEN
		RAISE EXCEPTION 'EMAIL DESCONOCIDO';
	END IF;
-- 3. Retornar Datos del User
	RETURN QUERY
	SELECT
	A.IDADMIN,
	A.NOMBRE,
	A.EMAIL,
	A.PWD
	FROM ADMINISTRADOR AS A
	WHERE A.IDADMIN = v_verificador;
END;
$$;


ALTER FUNCTION public.f_login_admin(v_email character varying) OWNER TO postgres;

--
-- TOC entry 237 (class 1255 OID 16968)
-- Name: f_login_cliente(character varying); Type: FUNCTION; Schema: public; Owner: postgres
--

CREATE FUNCTION public.f_login_cliente(v_email character varying) RETURNS TABLE(idcliente integer, nombre character varying, email character varying, pwd character varying)
    LANGUAGE plpgsql
    AS $$
DECLARE
v_verificador INTEGER;
BEGIN
-- 1. Consultar Email 
	SELECT C.IDCLIENTE
	INTO v_verificador
	FROM CLIENTE AS C
	WHERE C.EMAIL = v_email;
-- 2. Validar Si Existe el Cliente
	IF v_verificador IS NULL THEN
		RAISE EXCEPTION 'EMAIL DESCONOCIDO';
	END IF;
-- 3. Retornar Datos del User
	RETURN QUERY
	SELECT
	C.IDCLIENTE,
	C.NOMBRE,
	C.EMAIL,
	C.PWD
	FROM CLIENTE AS C
	WHERE C.IDCLIENTE = v_verificador;
END;
$$;


ALTER FUNCTION public.f_login_cliente(v_email character varying) OWNER TO postgres;

--
-- TOC entry 255 (class 1255 OID 16971)
-- Name: f_registar_cliente(character varying, character varying, character varying); Type: FUNCTION; Schema: public; Owner: postgres
--

CREATE FUNCTION public.f_registar_cliente(v_nombre character varying, v_email character varying, v_pwd character varying) RETURNS TABLE(idcliente integer, nombre character varying, email character varying, pwd character varying)
    LANGUAGE plpgsql
    AS $$
DECLARE
	v_verificador INTEGER;
BEGIN
	-- 1. Validar Correo
	SELECT C.IDCLIENTE
	INTO v_verificador
	FROM CLIENTE AS C
	WHERE C.EMAIL = v_email;
	
	IF v_verificador IS NOT NULL THEN
		RAISE EXCEPTION 'EMAIL EN USO';
	END IF;
	-- 2. Registrar Cliente Y Retornar IDCLIENTE
	INSERT INTO CLIENTE(NOMBRE,EMAIL,PWD)
	VALUES(v_nombre,v_email,v_pwd)
	RETURNING CLIENTE.IDCLIENTE
	INTO v_verificador;
	-- 3. Retornar Datos del Cliente
	RETURN QUERY
	SELECT
	C.IDCLIENTE AS IDCLIENTE,
	C.NOMBRE AS NOMBRE,
	C.EMAIL AS EMAIL,
	C.PWD AS PWD
	FROM CLIENTE AS C
	WHERE C.IDCLIENTE = v_verificador;
END;
$$;


ALTER FUNCTION public.f_registar_cliente(v_nombre character varying, v_email character varying, v_pwd character varying) OWNER TO postgres;

--
-- TOC entry 243 (class 1255 OID 16972)
-- Name: f_register_carrito(); Type: FUNCTION; Schema: public; Owner: postgres
--

CREATE FUNCTION public.f_register_carrito() RETURNS trigger
    LANGUAGE plpgsql
    AS $$
BEGIN
	INSERT INTO CARRITO_COMPRA(FkCliente)
	VALUES (NEW.IDCLIENTE);
	RETURN NEW;
END;
$$;


ALTER FUNCTION public.f_register_carrito() OWNER TO postgres;

--
-- TOC entry 239 (class 1255 OID 16980)
-- Name: f_update_carrito(); Type: FUNCTION; Schema: public; Owner: postgres
--

CREATE FUNCTION public.f_update_carrito() RETURNS trigger
    LANGUAGE plpgsql
    AS $$
BEGIN
	UPDATE CARRITO_COMPRA
	SET TOTAL = TOTAL + NEW.SubTotal
	WHERE IDCARRITO = NEW.FkCarrito;
	RETURN NEW;
END;
$$;


ALTER FUNCTION public.f_update_carrito() OWNER TO postgres;

--
-- TOC entry 242 (class 1255 OID 16995)
-- Name: p_delete_producto(integer); Type: PROCEDURE; Schema: public; Owner: postgres
--

CREATE PROCEDURE public.p_delete_producto(IN p_idproducto integer)
    LANGUAGE plpgsql
    AS $$
BEGIN
	IF p_idProducto <= 0 THEN
		RAISE EXCEPTION 'Porfavor Ingresar Un Id Valido';
	END IF;
	DELETE FROM PRODUCTO
	WHERE IDPRODUCTO = p_idProducto;
END;
$$;


ALTER PROCEDURE public.p_delete_producto(IN p_idproducto integer) OWNER TO postgres;

--
-- TOC entry 240 (class 1255 OID 16993)
-- Name: p_insert_productos(character varying, integer, numeric); Type: PROCEDURE; Schema: public; Owner: postgres
--

CREATE PROCEDURE public.p_insert_productos(IN p_nombre character varying, IN p_stock integer, IN p_precio numeric)
    LANGUAGE plpgsql
    AS $$
BEGIN
	IF p_stock < 0 THEN
		RAISE EXCEPTION 'Stock Debe ser Mayor o Igual a 0';
	ELSIF p_precio <= 0 THEN
		RAISE EXCEPTION 'Precio debe ser mayor a 0';
	END IF;
	INSERT INTO PRODUCTO(NOMBRE, STOCK, PRECIO)
	VALUES(p_nombre, p_stock, p_precio);
END;
$$;


ALTER PROCEDURE public.p_insert_productos(IN p_nombre character varying, IN p_stock integer, IN p_precio numeric) OWNER TO postgres;

--
-- TOC entry 241 (class 1255 OID 16994)
-- Name: p_update_producto(integer, integer, numeric); Type: PROCEDURE; Schema: public; Owner: postgres
--

CREATE PROCEDURE public.p_update_producto(IN p_idproducto integer, IN p_stock integer, IN p_precio numeric)
    LANGUAGE plpgsql
    AS $$
BEGIN
	IF p_stock < 0 THEN
		RAISE EXCEPTION 'Stock Debe ser Mayor o Igual a 0';
	ELSIF p_precio <= 0 THEN
		RAISE EXCEPTION 'Precio debe ser mayor a 0';
	END IF;
	UPDATE PRODUCTO
	SET STOCK = p_stock,
	PRECIO = p_precio
	WHERE IDPRODUCTO = p_idProducto;
END;
$$;


ALTER PROCEDURE public.p_update_producto(IN p_idproducto integer, IN p_stock integer, IN p_precio numeric) OWNER TO postgres;

--
-- TOC entry 258 (class 1255 OID 16977)
-- Name: registrar_venta(integer, integer[], integer[]); Type: PROCEDURE; Schema: public; Owner: postgres
--

CREATE PROCEDURE public.registrar_venta(IN p_idcliente integer, IN p_productos_id integer[], IN p_cantidad integer[])
    LANGUAGE plpgsql
    AS $$
DECLARE 
v_idventa INTEGER;
v_producto_id INTEGER;
v_nombre VARCHAR(100);
v_valor_unidad NUMERIC(10,2);
v_cantidad INTEGER;
v_stock INTEGER;
v_sub_total NUMERIC(10,2);
v_total NUMERIC(10,2);
v_indice INTEGER;
BEGIN
	-- Validar Igualdad de Longitud de Arrays
	IF array_length(p_productos_id, 1) <> array_length(p_cantidad, 1) THEN
		RAISE EXCEPTION 'Incongruencia de Longitud';
	END IF;
	
	-- INSERTAR VENTA
	INSERT INTO VENTA(FKCLIENTE)
	VALUES(p_idcliente)
	-- RETURNING, Util para Devolver Valores Recien Creados
	RETURNING IDVENTA
	INTO v_idventa;

	-- Objetivo, Recorrer todos Los Productos, Cantidades, Verificar Stock, Insertar en Detalles_Venta
	-- Los Arrays en Postgres Empiezan en 1, NO en 0
	-- SELECT GENERATE_SUBSCRIPTS(p_productos_id, 1), Consultar el Valor e Indice del Array p_productos_id
	FOR v_indice IN (SELECT GENERATE_SUBSCRIPTS(p_productos_id, 1))
	LOOP
		
		-- Asignar a Variables, El Id Producto y Cantidad Correspondiente
		v_producto_id := p_productos_id[v_indice];
		v_cantidad := p_cantidad[v_indice];
		
		-- Query Para Asignar Valores Correspondientes del Producto
			SELECT 
			p.NOMBRE, p.PRECIO, p.STOCK
			INTO
			v_nombre, v_valor_unidad, v_stock
			FROM
			PRODUCTO AS p
			WHERE 
			p.IDPRODUCTO = v_producto_id;
			
		-- Verificador De Stock Disponible
			IF v_stock < v_cantidad THEN
				RAISE EXCEPTION 'STOCK INSUFICIENTE DE, %, Unidades Restantes, %', v_nombre, v_stock;
			END IF;
			
		-- Sumatoria Sub_valor
		v_sub_total := v_valor_unidad * v_cantidad;
	
		-- Registrar en Detalle Venta
		INSERT INTO DETALLE_VENTA(FKVENTA, FKPRODUCTO, CANTIDAD, SUBTOTAL)
		VALUES(v_idventa, v_producto_id, v_cantidad, v_sub_total);

		
	END LOOP;

	-- Calcular Valor Total de Compra
	SELECT SUM(DV.SUBTOTAL)
	INTO v_total 
	FROM DETALLE_VENTA AS DV
	WHERE DV.FKVENTA = v_idventa;

	-- Actualizar Valor Venta
	UPDATE VENTA
	SET TOTAL = v_total
	WHERE IDVENTA = v_idventa;

END;
$$;


ALTER PROCEDURE public.registrar_venta(IN p_idcliente integer, IN p_productos_id integer[], IN p_cantidad integer[]) OWNER TO postgres;

SET default_tablespace = '';

SET default_table_access_method = heap;

--
-- TOC entry 220 (class 1259 OID 16892)
-- Name: administrador; Type: TABLE; Schema: public; Owner: postgres
--

CREATE TABLE public.administrador (
    idadmin integer NOT NULL,
    nombre character varying(100) NOT NULL,
    email character varying(250) NOT NULL,
    pwd character varying(250) NOT NULL
);


ALTER TABLE public.administrador OWNER TO postgres;

--
-- TOC entry 219 (class 1259 OID 16891)
-- Name: administrador_idadmin_seq; Type: SEQUENCE; Schema: public; Owner: postgres
--

CREATE SEQUENCE public.administrador_idadmin_seq
    AS integer
    START WITH 1
    INCREMENT BY 1
    NO MINVALUE
    NO MAXVALUE
    CACHE 1;


ALTER SEQUENCE public.administrador_idadmin_seq OWNER TO postgres;

--
-- TOC entry 5018 (class 0 OID 0)
-- Dependencies: 219
-- Name: administrador_idadmin_seq; Type: SEQUENCE OWNED BY; Schema: public; Owner: postgres
--

ALTER SEQUENCE public.administrador_idadmin_seq OWNED BY public.administrador.idadmin;


--
-- TOC entry 224 (class 1259 OID 16910)
-- Name: carrito_compra; Type: TABLE; Schema: public; Owner: postgres
--

CREATE TABLE public.carrito_compra (
    idcarrito integer NOT NULL,
    fkcliente integer NOT NULL,
    total numeric(10,2) DEFAULT 0
);


ALTER TABLE public.carrito_compra OWNER TO postgres;

--
-- TOC entry 223 (class 1259 OID 16909)
-- Name: carrito_compra_idcarrito_seq; Type: SEQUENCE; Schema: public; Owner: postgres
--

CREATE SEQUENCE public.carrito_compra_idcarrito_seq
    AS integer
    START WITH 1
    INCREMENT BY 1
    NO MINVALUE
    NO MAXVALUE
    CACHE 1;


ALTER SEQUENCE public.carrito_compra_idcarrito_seq OWNER TO postgres;

--
-- TOC entry 5019 (class 0 OID 0)
-- Dependencies: 223
-- Name: carrito_compra_idcarrito_seq; Type: SEQUENCE OWNED BY; Schema: public; Owner: postgres
--

ALTER SEQUENCE public.carrito_compra_idcarrito_seq OWNED BY public.carrito_compra.idcarrito;


--
-- TOC entry 218 (class 1259 OID 16881)
-- Name: cliente; Type: TABLE; Schema: public; Owner: postgres
--

CREATE TABLE public.cliente (
    idcliente integer NOT NULL,
    nombre character varying(100) NOT NULL,
    email character varying(250) NOT NULL,
    pwd character varying(250)
);


ALTER TABLE public.cliente OWNER TO postgres;

--
-- TOC entry 217 (class 1259 OID 16880)
-- Name: cliente_idcliente_seq; Type: SEQUENCE; Schema: public; Owner: postgres
--

CREATE SEQUENCE public.cliente_idcliente_seq
    AS integer
    START WITH 1
    INCREMENT BY 1
    NO MINVALUE
    NO MAXVALUE
    CACHE 1;


ALTER SEQUENCE public.cliente_idcliente_seq OWNER TO postgres;

--
-- TOC entry 5020 (class 0 OID 0)
-- Dependencies: 217
-- Name: cliente_idcliente_seq; Type: SEQUENCE OWNED BY; Schema: public; Owner: postgres
--

ALTER SEQUENCE public.cliente_idcliente_seq OWNED BY public.cliente.idcliente;


--
-- TOC entry 226 (class 1259 OID 16922)
-- Name: detalle_carrito; Type: TABLE; Schema: public; Owner: postgres
--

CREATE TABLE public.detalle_carrito (
    iddetallecarrito integer NOT NULL,
    fkcarrito integer NOT NULL,
    fkproducto integer NOT NULL,
    cantidad integer NOT NULL,
    subtotal numeric(10,2) NOT NULL
);


ALTER TABLE public.detalle_carrito OWNER TO postgres;

--
-- TOC entry 225 (class 1259 OID 16921)
-- Name: detalle_carrito_iddetallecarrito_seq; Type: SEQUENCE; Schema: public; Owner: postgres
--

CREATE SEQUENCE public.detalle_carrito_iddetallecarrito_seq
    AS integer
    START WITH 1
    INCREMENT BY 1
    NO MINVALUE
    NO MAXVALUE
    CACHE 1;


ALTER SEQUENCE public.detalle_carrito_iddetallecarrito_seq OWNER TO postgres;

--
-- TOC entry 5021 (class 0 OID 0)
-- Dependencies: 225
-- Name: detalle_carrito_iddetallecarrito_seq; Type: SEQUENCE OWNED BY; Schema: public; Owner: postgres
--

ALTER SEQUENCE public.detalle_carrito_iddetallecarrito_seq OWNED BY public.detalle_carrito.iddetallecarrito;


--
-- TOC entry 230 (class 1259 OID 16952)
-- Name: detalle_venta; Type: TABLE; Schema: public; Owner: postgres
--

CREATE TABLE public.detalle_venta (
    iddetalleventa integer NOT NULL,
    fkventa integer NOT NULL,
    fkproducto integer NOT NULL,
    cantidad integer NOT NULL,
    subtotal numeric(10,2) NOT NULL
);


ALTER TABLE public.detalle_venta OWNER TO postgres;

--
-- TOC entry 229 (class 1259 OID 16951)
-- Name: detalle_venta_iddetalleventa_seq; Type: SEQUENCE; Schema: public; Owner: postgres
--

CREATE SEQUENCE public.detalle_venta_iddetalleventa_seq
    AS integer
    START WITH 1
    INCREMENT BY 1
    NO MINVALUE
    NO MAXVALUE
    CACHE 1;


ALTER SEQUENCE public.detalle_venta_iddetalleventa_seq OWNER TO postgres;

--
-- TOC entry 5022 (class 0 OID 0)
-- Dependencies: 229
-- Name: detalle_venta_iddetalleventa_seq; Type: SEQUENCE OWNED BY; Schema: public; Owner: postgres
--

ALTER SEQUENCE public.detalle_venta_iddetalleventa_seq OWNED BY public.detalle_venta.iddetalleventa;


--
-- TOC entry 234 (class 1259 OID 17012)
-- Name: info_cliente; Type: VIEW; Schema: public; Owner: postgres
--

CREATE VIEW public.info_cliente AS
SELECT
    NULL::character varying(100) AS cliente,
    NULL::bigint AS ventas_realizadas,
    NULL::bigint AS productos_comprados,
    NULL::numeric AS total;


ALTER VIEW public.info_cliente OWNER TO postgres;

--
-- TOC entry 222 (class 1259 OID 16903)
-- Name: producto; Type: TABLE; Schema: public; Owner: postgres
--

CREATE TABLE public.producto (
    idproducto integer NOT NULL,
    nombre character varying(100) NOT NULL,
    stock integer NOT NULL,
    precio numeric(10,2) NOT NULL
);


ALTER TABLE public.producto OWNER TO postgres;

--
-- TOC entry 221 (class 1259 OID 16902)
-- Name: producto_idproducto_seq; Type: SEQUENCE; Schema: public; Owner: postgres
--

CREATE SEQUENCE public.producto_idproducto_seq
    AS integer
    START WITH 1
    INCREMENT BY 1
    NO MINVALUE
    NO MAXVALUE
    CACHE 1;


ALTER SEQUENCE public.producto_idproducto_seq OWNER TO postgres;

--
-- TOC entry 5023 (class 0 OID 0)
-- Dependencies: 221
-- Name: producto_idproducto_seq; Type: SEQUENCE OWNED BY; Schema: public; Owner: postgres
--

ALTER SEQUENCE public.producto_idproducto_seq OWNED BY public.producto.idproducto;


--
-- TOC entry 235 (class 1259 OID 17017)
-- Name: promedio_cliente; Type: VIEW; Schema: public; Owner: postgres
--

CREATE VIEW public.promedio_cliente AS
SELECT
    NULL::character varying(100) AS cliente,
    NULL::bigint AS compras_realizadas,
    NULL::numeric AS gasto_x_compra,
    NULL::numeric AS productos_x_compra;


ALTER VIEW public.promedio_cliente OWNER TO postgres;

--
-- TOC entry 228 (class 1259 OID 16939)
-- Name: venta; Type: TABLE; Schema: public; Owner: postgres
--

CREATE TABLE public.venta (
    idventa integer NOT NULL,
    fkcliente integer NOT NULL,
    fecha date DEFAULT CURRENT_DATE,
    total numeric(10,2)
);


ALTER TABLE public.venta OWNER TO postgres;

--
-- TOC entry 233 (class 1259 OID 17008)
-- Name: tendencias_by_doy; Type: VIEW; Schema: public; Owner: postgres
--

CREATE VIEW public.tendencias_by_doy AS
 SELECT date_part('doy'::text, v.fecha) AS dia,
    sum(dv.cantidad) AS cantidades_vendidas,
    sum(dv.subtotal) AS total
   FROM (public.detalle_venta dv
     JOIN public.venta v ON ((dv.fkventa = v.idventa)))
  GROUP BY (date_part('doy'::text, v.fecha));


ALTER VIEW public.tendencias_by_doy OWNER TO postgres;

--
-- TOC entry 232 (class 1259 OID 17004)
-- Name: tendencias_by_month; Type: VIEW; Schema: public; Owner: postgres
--

CREATE VIEW public.tendencias_by_month AS
 SELECT date_part('month'::text, v.fecha) AS dia,
    sum(dv.cantidad) AS cantidades_vendidas,
    sum(dv.subtotal) AS total
   FROM (public.detalle_venta dv
     JOIN public.venta v ON ((dv.fkventa = v.idventa)))
  GROUP BY (date_part('month'::text, v.fecha));


ALTER VIEW public.tendencias_by_month OWNER TO postgres;

--
-- TOC entry 231 (class 1259 OID 17000)
-- Name: tendencias_productos; Type: VIEW; Schema: public; Owner: postgres
--

CREATE VIEW public.tendencias_productos AS
SELECT
    NULL::character varying(100) AS nombre,
    NULL::bigint AS cantidades_vendidas,
    NULL::numeric AS total;


ALTER VIEW public.tendencias_productos OWNER TO postgres;

--
-- TOC entry 236 (class 1259 OID 17026)
-- Name: top_productos; Type: VIEW; Schema: public; Owner: postgres
--

CREATE VIEW public.top_productos AS
SELECT
    NULL::character varying(100) AS nombre,
    NULL::bigint AS stock_vendido,
    NULL::numeric AS ventas_generadas;


ALTER VIEW public.top_productos OWNER TO postgres;

--
-- TOC entry 227 (class 1259 OID 16938)
-- Name: venta_idventa_seq; Type: SEQUENCE; Schema: public; Owner: postgres
--

CREATE SEQUENCE public.venta_idventa_seq
    AS integer
    START WITH 1
    INCREMENT BY 1
    NO MINVALUE
    NO MAXVALUE
    CACHE 1;


ALTER SEQUENCE public.venta_idventa_seq OWNER TO postgres;

--
-- TOC entry 5024 (class 0 OID 0)
-- Dependencies: 227
-- Name: venta_idventa_seq; Type: SEQUENCE OWNED BY; Schema: public; Owner: postgres
--

ALTER SEQUENCE public.venta_idventa_seq OWNED BY public.venta.idventa;


--
-- TOC entry 4810 (class 2604 OID 16895)
-- Name: administrador idadmin; Type: DEFAULT; Schema: public; Owner: postgres
--

ALTER TABLE ONLY public.administrador ALTER COLUMN idadmin SET DEFAULT nextval('public.administrador_idadmin_seq'::regclass);


--
-- TOC entry 4812 (class 2604 OID 16913)
-- Name: carrito_compra idcarrito; Type: DEFAULT; Schema: public; Owner: postgres
--

ALTER TABLE ONLY public.carrito_compra ALTER COLUMN idcarrito SET DEFAULT nextval('public.carrito_compra_idcarrito_seq'::regclass);


--
-- TOC entry 4809 (class 2604 OID 16884)
-- Name: cliente idcliente; Type: DEFAULT; Schema: public; Owner: postgres
--

ALTER TABLE ONLY public.cliente ALTER COLUMN idcliente SET DEFAULT nextval('public.cliente_idcliente_seq'::regclass);


--
-- TOC entry 4814 (class 2604 OID 16925)
-- Name: detalle_carrito iddetallecarrito; Type: DEFAULT; Schema: public; Owner: postgres
--

ALTER TABLE ONLY public.detalle_carrito ALTER COLUMN iddetallecarrito SET DEFAULT nextval('public.detalle_carrito_iddetallecarrito_seq'::regclass);


--
-- TOC entry 4817 (class 2604 OID 16955)
-- Name: detalle_venta iddetalleventa; Type: DEFAULT; Schema: public; Owner: postgres
--

ALTER TABLE ONLY public.detalle_venta ALTER COLUMN iddetalleventa SET DEFAULT nextval('public.detalle_venta_iddetalleventa_seq'::regclass);


--
-- TOC entry 4811 (class 2604 OID 16906)
-- Name: producto idproducto; Type: DEFAULT; Schema: public; Owner: postgres
--

ALTER TABLE ONLY public.producto ALTER COLUMN idproducto SET DEFAULT nextval('public.producto_idproducto_seq'::regclass);


--
-- TOC entry 4815 (class 2604 OID 16942)
-- Name: venta idventa; Type: DEFAULT; Schema: public; Owner: postgres
--

ALTER TABLE ONLY public.venta ALTER COLUMN idventa SET DEFAULT nextval('public.venta_idventa_seq'::regclass);



--
-- TOC entry 5025 (class 0 OID 0)
-- Dependencies: 219
-- Name: administrador_idadmin_seq; Type: SEQUENCE SET; Schema: public; Owner: postgres
--

SELECT pg_catalog.setval('public.administrador_idadmin_seq', 1, true);


--
-- TOC entry 5026 (class 0 OID 0)
-- Dependencies: 223
-- Name: carrito_compra_idcarrito_seq; Type: SEQUENCE SET; Schema: public; Owner: postgres
--

SELECT pg_catalog.setval('public.carrito_compra_idcarrito_seq', 15, true);


--
-- TOC entry 5027 (class 0 OID 0)
-- Dependencies: 217
-- Name: cliente_idcliente_seq; Type: SEQUENCE SET; Schema: public; Owner: postgres
--

SELECT pg_catalog.setval('public.cliente_idcliente_seq', 15, true);


--
-- TOC entry 5028 (class 0 OID 0)
-- Dependencies: 225
-- Name: detalle_carrito_iddetallecarrito_seq; Type: SEQUENCE SET; Schema: public; Owner: postgres
--

SELECT pg_catalog.setval('public.detalle_carrito_iddetallecarrito_seq', 23, true);


--
-- TOC entry 5029 (class 0 OID 0)
-- Dependencies: 229
-- Name: detalle_venta_iddetalleventa_seq; Type: SEQUENCE SET; Schema: public; Owner: postgres
--

SELECT pg_catalog.setval('public.detalle_venta_iddetalleventa_seq', 17, true);


--
-- TOC entry 5030 (class 0 OID 0)
-- Dependencies: 221
-- Name: producto_idproducto_seq; Type: SEQUENCE SET; Schema: public; Owner: postgres
--

SELECT pg_catalog.setval('public.producto_idproducto_seq', 6, true);


--
-- TOC entry 5031 (class 0 OID 0)
-- Dependencies: 227
-- Name: venta_idventa_seq; Type: SEQUENCE SET; Schema: public; Owner: postgres
--

SELECT pg_catalog.setval('public.venta_idventa_seq', 19, true);


--
-- TOC entry 4823 (class 2606 OID 16901)
-- Name: administrador administrador_email_key; Type: CONSTRAINT; Schema: public; Owner: postgres
--

ALTER TABLE ONLY public.administrador
    ADD CONSTRAINT administrador_email_key UNIQUE (email);


--
-- TOC entry 4825 (class 2606 OID 16899)
-- Name: administrador administrador_pkey; Type: CONSTRAINT; Schema: public; Owner: postgres
--

ALTER TABLE ONLY public.administrador
    ADD CONSTRAINT administrador_pkey PRIMARY KEY (idadmin);


--
-- TOC entry 4829 (class 2606 OID 16915)
-- Name: carrito_compra carrito_compra_pkey; Type: CONSTRAINT; Schema: public; Owner: postgres
--

ALTER TABLE ONLY public.carrito_compra
    ADD CONSTRAINT carrito_compra_pkey PRIMARY KEY (idcarrito);


--
-- TOC entry 4819 (class 2606 OID 16890)
-- Name: cliente cliente_email_key; Type: CONSTRAINT; Schema: public; Owner: postgres
--

ALTER TABLE ONLY public.cliente
    ADD CONSTRAINT cliente_email_key UNIQUE (email);


--
-- TOC entry 4821 (class 2606 OID 16888)
-- Name: cliente cliente_pkey; Type: CONSTRAINT; Schema: public; Owner: postgres
--

ALTER TABLE ONLY public.cliente
    ADD CONSTRAINT cliente_pkey PRIMARY KEY (idcliente);


--
-- TOC entry 4831 (class 2606 OID 16927)
-- Name: detalle_carrito detalle_carrito_pkey; Type: CONSTRAINT; Schema: public; Owner: postgres
--

ALTER TABLE ONLY public.detalle_carrito
    ADD CONSTRAINT detalle_carrito_pkey PRIMARY KEY (iddetallecarrito);


--
-- TOC entry 4835 (class 2606 OID 16957)
-- Name: detalle_venta detalle_venta_pkey; Type: CONSTRAINT; Schema: public; Owner: postgres
--

ALTER TABLE ONLY public.detalle_venta
    ADD CONSTRAINT detalle_venta_pkey PRIMARY KEY (iddetalleventa);


--
-- TOC entry 4827 (class 2606 OID 16908)
-- Name: producto producto_pkey; Type: CONSTRAINT; Schema: public; Owner: postgres
--

ALTER TABLE ONLY public.producto
    ADD CONSTRAINT producto_pkey PRIMARY KEY (idproducto);


--
-- TOC entry 4833 (class 2606 OID 16945)
-- Name: venta venta_pkey; Type: CONSTRAINT; Schema: public; Owner: postgres
--

ALTER TABLE ONLY public.venta
    ADD CONSTRAINT venta_pkey PRIMARY KEY (idventa);


--
-- TOC entry 4992 (class 2618 OID 17003)
-- Name: tendencias_productos _RETURN; Type: RULE; Schema: public; Owner: postgres
--

CREATE OR REPLACE VIEW public.tendencias_productos AS
 SELECT p.nombre,
    sum(dv.cantidad) AS cantidades_vendidas,
    sum(dv.subtotal) AS total
   FROM (public.detalle_venta dv
     JOIN public.producto p ON ((dv.fkproducto = p.idproducto)))
  GROUP BY p.idproducto;


--
-- TOC entry 4995 (class 2618 OID 17015)
-- Name: info_cliente _RETURN; Type: RULE; Schema: public; Owner: postgres
--

CREATE OR REPLACE VIEW public.info_cliente AS
 SELECT c.nombre AS cliente,
    count(v.idventa) AS ventas_realizadas,
    sum(dv.cantidad) AS productos_comprados,
    sum(dv.subtotal) AS total
   FROM ((public.detalle_venta dv
     JOIN public.venta v ON ((dv.fkventa = v.idventa)))
     JOIN public.cliente c ON ((v.fkcliente = c.idcliente)))
  GROUP BY c.idcliente;


--
-- TOC entry 4996 (class 2618 OID 17020)
-- Name: promedio_cliente _RETURN; Type: RULE; Schema: public; Owner: postgres
--

CREATE OR REPLACE VIEW public.promedio_cliente AS
 SELECT c.nombre AS cliente,
    count(v.idventa) AS compras_realizadas,
    trunc(avg(dv.subtotal), 2) AS gasto_x_compra,
    trunc(avg(dv.cantidad), 2) AS productos_x_compra
   FROM ((public.detalle_venta dv
     JOIN public.venta v ON ((dv.fkventa = v.idventa)))
     JOIN public.cliente c ON ((v.fkcliente = c.idcliente)))
  GROUP BY c.idcliente;


--
-- TOC entry 4997 (class 2618 OID 17029)
-- Name: top_productos _RETURN; Type: RULE; Schema: public; Owner: postgres
--

CREATE OR REPLACE VIEW public.top_productos AS
 SELECT p.nombre,
    sum(dv.cantidad) AS stock_vendido,
    sum(dv.subtotal) AS ventas_generadas
   FROM (public.detalle_venta dv
     JOIN public.producto p ON ((dv.fkproducto = p.idproducto)))
  GROUP BY p.idproducto
  ORDER BY (sum(dv.cantidad)) DESC;


--
-- TOC entry 4843 (class 2620 OID 16981)
-- Name: detalle_carrito t_after_insert_detalle_carrito; Type: TRIGGER; Schema: public; Owner: postgres
--

CREATE TRIGGER t_after_insert_detalle_carrito AFTER INSERT ON public.detalle_carrito FOR EACH ROW EXECUTE FUNCTION public.f_update_carrito();


--
-- TOC entry 4844 (class 2620 OID 16990)
-- Name: detalle_carrito t_after_update_detalle_carrito; Type: TRIGGER; Schema: public; Owner: postgres
--

CREATE TRIGGER t_after_update_detalle_carrito AFTER UPDATE ON public.detalle_carrito FOR EACH ROW EXECUTE FUNCTION public.f_after_update_detalle_carrito();


--
-- TOC entry 4845 (class 2620 OID 16984)
-- Name: detalle_venta t_delete_detalle_carrito; Type: TRIGGER; Schema: public; Owner: postgres
--

CREATE TRIGGER t_delete_detalle_carrito AFTER INSERT ON public.detalle_venta FOR EACH ROW EXECUTE FUNCTION public.f_delete_detalle_carrito();


--
-- TOC entry 4842 (class 2620 OID 16973)
-- Name: cliente trigger_after_insert_cliente; Type: TRIGGER; Schema: public; Owner: postgres
--

CREATE TRIGGER trigger_after_insert_cliente AFTER INSERT ON public.cliente FOR EACH ROW EXECUTE FUNCTION public.f_register_carrito();


--
-- TOC entry 4846 (class 2620 OID 16979)
-- Name: detalle_venta trigger_update_stock; Type: TRIGGER; Schema: public; Owner: postgres
--

CREATE TRIGGER trigger_update_stock AFTER INSERT ON public.detalle_venta FOR EACH ROW EXECUTE FUNCTION public.f_actualizar_stock();


--
-- TOC entry 4836 (class 2606 OID 16916)
-- Name: carrito_compra carrito_compra_fkcliente_fkey; Type: FK CONSTRAINT; Schema: public; Owner: postgres
--

ALTER TABLE ONLY public.carrito_compra
    ADD CONSTRAINT carrito_compra_fkcliente_fkey FOREIGN KEY (fkcliente) REFERENCES public.cliente(idcliente);


--
-- TOC entry 4837 (class 2606 OID 16928)
-- Name: detalle_carrito detalle_carrito_fkcarrito_fkey; Type: FK CONSTRAINT; Schema: public; Owner: postgres
--

ALTER TABLE ONLY public.detalle_carrito
    ADD CONSTRAINT detalle_carrito_fkcarrito_fkey FOREIGN KEY (fkcarrito) REFERENCES public.carrito_compra(idcarrito);


--
-- TOC entry 4838 (class 2606 OID 16933)
-- Name: detalle_carrito detalle_carrito_fkproducto_fkey; Type: FK CONSTRAINT; Schema: public; Owner: postgres
--

ALTER TABLE ONLY public.detalle_carrito
    ADD CONSTRAINT detalle_carrito_fkproducto_fkey FOREIGN KEY (fkproducto) REFERENCES public.producto(idproducto);


--
-- TOC entry 4840 (class 2606 OID 16963)
-- Name: detalle_venta detalle_venta_fkproducto_fkey; Type: FK CONSTRAINT; Schema: public; Owner: postgres
--

ALTER TABLE ONLY public.detalle_venta
    ADD CONSTRAINT detalle_venta_fkproducto_fkey FOREIGN KEY (fkproducto) REFERENCES public.producto(idproducto);


--
-- TOC entry 4841 (class 2606 OID 16958)
-- Name: detalle_venta detalle_venta_fkventa_fkey; Type: FK CONSTRAINT; Schema: public; Owner: postgres
--

ALTER TABLE ONLY public.detalle_venta
    ADD CONSTRAINT detalle_venta_fkventa_fkey FOREIGN KEY (fkventa) REFERENCES public.venta(idventa);


--
-- TOC entry 4839 (class 2606 OID 16946)
-- Name: venta venta_fkcliente_fkey; Type: FK CONSTRAINT; Schema: public; Owner: postgres
--

ALTER TABLE ONLY public.venta
    ADD CONSTRAINT venta_fkcliente_fkey FOREIGN KEY (fkcliente) REFERENCES public.cliente(idcliente);


-- Completed on 2025-09-30 21:42:09

--
-- PostgreSQL database dump complete
--

