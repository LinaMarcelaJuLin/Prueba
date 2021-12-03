# Prueba
import datetime
import json
import pytz
from django.db.models import Q, F, Max

from domain.api.serializers import (
    FranjaHorarioSerializer,QuestionServiceSerializer,FranjaHorarioRroveedoresSerializer, 
    DetalleAsistenciaProgramadaSerializer, AfiliadoUbigeoSerializer, EstadoProveedorSerializer,
    RegistroAfiliadoSerializer
)
from django.http import HttpResponse
from django.views.generic import View
from django.db import connections
from rest_framework.permissions import AllowAny, IsAuthenticated
from rest_framework.generics import ListAPIView, ListCreateAPIView, CreateAPIView
from rest_framework.mixins import UpdateModelMixin
from rest_framework.response import Response
from rest_framework import exceptions
from rest_framework import views


from domain.api.helpers import (
    obtener_franja_servicio, notificaciones_base_proveedores, get_zona_horaria, 
    get_idpais, registro_afiliados
)
from domain.api.authentications import TokenAuthentication, MyTokenAuthentication
from domain.api.task import envio_notificaciones_asc
from domain.api.utils import ObtenerTiemposDistancia, DistanciaDosPuntos
from domain.catalogo.helpers import *
from domain.catalogo.models import (
    CatalogoServicio, CatalogoProveedor, HorariosServicio, CatalogoMetadataServicio,
    CatalogoMetadataServicioOptionValue, CatalogoServicioAsistenciaProveedor, CatalogoProgramaServicio
)
from api_apps.settings.routers import AuthRouter
from domain.temporal.helpers import *
from domain.temporal.models import (
    Asistencia,CitasProgramadas,AsistenciasRechazadas,
    AsistenciasIgnoradas, AsistenciaUbigeoAfiliado, AsistenciaUbigeoBeneficiarios,
    AsistenciaAsigProveedor, AsistenciaUbigeoProveedor, AsistenciasEncoladas,InfoPasarelaPago
)
#Detalle de la Asistencia Programada del Afiliado (flujo programadas) 
class DetalleAsistenciaProgramadaAfiliadoView(views.APIView):
    name = 'detail-assist-affiliate'
    authentication_classes = (MyTokenAuthentication,)

    def get(self, request):
        serilizer = DetalleAsistenciaProgramadaSerializer(data=request.GET)
        if serilizer.is_valid():
            pais  = serilizer.data['country'].lower()
            idasistencia  = serilizer.data['idasistencia']
            catalogo = connections['soaang_' + pais + "_catalogo"].cursor()
            asistencia = GetAsistencia(idasistencia, pais)

            if asistencia:
                asistencia_programada = GetAsistenciasProgramadas(idasistencia, pais)
                if asistencia_programada:
                    proveedor_asignado = GetAsistenciaAsigProveedor(idasistencia, pais)
                    if proveedor_asignado:
                        query = """
                        SELECT cs.DESCRIPCION,  cs.MIN_RECORRIDO, cs.DURACION_SERVICIO, cs.MIN_CANCELADO 
                        FROM catalogo_proveedor_servicio cps inner join catalogo_servicio cs
                        ON cps.idservicio=cs.idservicio WHERE cps.idservicio=%s AND cps.idproveedor=%s
                        """
                        catalogo.execute(query, [asistencia.idservicio, proveedor_asignado.idproveedor])
                        rows = catalogo.fetchone()
                        if rows:
                            descripcion = rows[0]
                            min_recorrido = rows[1]
                            duracion_servicio = rows[2]
                            min_cancelado = rows[3]
                        else:
                            msg = {'message': 'No existe un proveedor que preste este servicio - ErrorCode: 0'}
                            return Response(msg, status=203)
                        query = """
                        SELECT cp.NOMBRECOMERCIAL, cpu.DIRECCION, cpu.LATITUD, cpu.LONGITUD, cpt.NUMEROTELEFONO FROM catalogo_proveedor cp INNER JOIN catalogo_proveedor_telefono cpt INNER JOIN catalogo_proveedor_ubigeo cpu
                        ON cp.idproveedor=cpu.idproveedor AND cp.idproveedor=cpt.idproveedor
                        WHERE cp.idproveedor=%s
                        """
                        catalogo.execute(query, [proveedor_asignado.idproveedor])
                        rows = catalogo.fetchone()
                        if rows:
                            nombre_proveedor = rows[0]
direccion = rows[1]
                            latitud = rows[2]
                            longitud = rows[3]
                            telefono = rows[4]
                        res = {
                            'nombre_proveedor': nombre_proveedor, 'nombre_servicio': descripcion, 'direccion': direccion, 'latitud': latitud,
                            'longitud': longitud, 'fecha_programada': asistencia_programada[0].fechaprogramada, 'min_recorrido': min_recorrido,
                            'diracion_servicio': duracion_servicio, 'min_cancelado': min_cancelado, 'telefono_proveedor': telefono
                        }
                        return Response(res)
                    else:
                        msg = {'message': 'No se ha asignado un proveedor a la asistencia - ErrorCode: 0'}
                else:
                    msg = {'message': 'No tiene asistencias programadas - ErrorCode: 0'}
            else:
                msg = {'message': 'La asistencia no existe - ErrorCode: 0'}
            return Response(msg, status=203)
        else:
            return Response(serilizer.errors)

    def http_method_not_allowed(self, request, *args, **kwargs):
        msg = (request.method)
        raise exceptions.MethodNotAllowed(msg)
